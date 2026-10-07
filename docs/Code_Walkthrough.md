# Code Walkthrough: Thrust Stand Firmware

This covers every piece of custom code you wrote (not the CubeMX-generated
boilerplate), explaining what each line does, why it's there, and what
would break without it.

---

## 1. The global variable

```c
volatile int32_t hx711_raw = 0;
```

- **`int32_t`**, a 32-bit signed integer. We need this size because the
  HX711 outputs a 24-bit signed value, and a 32-bit container is the
  smallest standard type that can hold a 24-bit signed number safely
  without special-casing.
- **`volatile`**, tells the compiler "this value can change outside of
  normal program flow, don't optimize it away." Without this, the compiler
  is technically allowed to assume the variable never changes between
  reads (since nothing *visible* in the code changes it externally) and
  could optimize your Live Expressions view into showing stale or
  incorrect data, or in worse cases optimize away code that reads it.
  **Without `volatile`:** the debugger might show you a cached/wrong value,
  or a more aggressive compiler setting could break the variable update
  entirely.
- **`= 0`**, gives it a starting value so it's not garbage memory before
  the first real read happens.

---

## 2. The `HX711_Read()` function

```c
static int32_t HX711_Read(void)
{
```
- **`static`**, limits this function's visibility to this file only. It's
  not meant to be called from anywhere else in your project, so this
  keeps your code organized and avoids naming conflicts. **Without it:**
  the code would still work, but the function would be visible/callable
  from other files unnecessarily, a minor cleanliness issue, not a
  functional one.
- **`int32_t` return type**, matches the type of value it's building and
  handing back to the caller.

```c
    int32_t value = 0;
```
- A local variable that accumulates the 24 bits we're about to read, one
  at a time. Starts at zero so we're not building on top of garbage.

```c
    while (HAL_GPIO_ReadPin(HX711_DT_GPIO_Port, HX711_DT_Pin) == GPIO_PIN_SET);
```
- This is the HX711's way of telling you "I have a new measurement ready."
  It pulls its DT pin LOW when data is ready; until then, it's HIGH.
  This line just spins in place, doing nothing, for as long as DT is
  HIGH, and falls through the instant it goes LOW.
- **Why necessary:** the HX711 takes a small but real amount of time
  internally to complete each measurement. If you read before it's ready,
  you'd get a partial or meaningless value. This is what real embedded
  code means by "polling", repeatedly checking a condition instead of
  doing something more efficient like an interrupt.
- **Without it:** you'd occasionally read garbage, because you'd be
  reading mid-measurement rather than waiting for a genuinely completed
  one. This was actually the exact bug behind your very first "stuck
  forever" symptom, when DT never went low (due to the wiring/power
  issue), this line spun forever with nothing after it ever executing.

```c
    for (int i = 0; i < 24; i++)
    {
```
- The HX711 sends its 24-bit result one bit at a time. This loop runs
  exactly 24 times to collect every bit. **Without the right count:**
  fewer than 24 gives you an incomplete, wrong value; more than 24 starts
  reading into the "extra pulse" territory meant for something else
  (explained below), corrupting your reading.

```c
        HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_SET);
        value = value << 1;
```
- Setting SCK HIGH is *you* telling the HX711 "give me the next bit."
- `value = value << 1` shifts every bit already in `value` one position
  to the left, making room at the bottom for the new bit you're about to
  read. This is how you build up a multi-bit number one bit at a time -
  think of it like writing digits left to right; each new digit pushes
  the existing ones over.
- **Without the shift:** every new bit would just overwrite position 0
  instead of building a real multi-bit number, you'd end up with only
  the very last bit read, not a meaningful 24-bit value.

```c
        if (HAL_GPIO_ReadPin(HX711_DT_GPIO_Port, HX711_DT_Pin) == GPIO_PIN_SET)
        {
            value++;
        }
```
- This reads the actual bit **while SCK is still HIGH**, this timing
  matters and was the exact bug you hit yesterday (reading after dropping
  SCK low gave you consistently wrong/zero values, since you were reading
  at the wrong point in the HX711's timing).
- If the bit is HIGH, `value++` sets the newly-shifted-in bottom bit to 1
  (since after the left-shift, the bottom bit is 0 by default, adding 1
  flips just that bottom bit). If it's LOW, we do nothing, leaving that
  bottom bit as 0.
- **Without correct timing here:** this is exactly what caused your
  0/511/1023/2047/4095 (all-ones-pattern) bugs, reading at the wrong
  moment reads garbage or a stuck value instead of the real bit.

```c
        HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_RESET);
    }
```
- Drops SCK back LOW, completing one full clock pulse. The HX711 will
  present the *next* bit on DT once SCK goes high again on the next loop
  iteration. **Without this:** SCK would stay high, and the HX711's timing
  would never advance to the next bit, you'd read the same bit 24 times.

```c
    HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_RESET);
```
- A 25th pulse. This isn't data, per the HX711's datasheet, this extra
  pulse tells the chip which mode/gain to use for the *next* reading
  (channel A, gain 128, in our case, the default and most common mode).
- **Without this pulse:** the HX711 can end up in an unexpected mode for
  future readings, or subsequent reads can become misaligned/unreliable,
  since the chip is expecting this pulse as part of its normal cycle.

```c
    if (value & 0x800000)
    {
        value |= 0xFF000000;
    }
```
- This is **sign extension**. The HX711 outputs a *signed* 24-bit number,
  meaning it can represent negative values (e.g., if your baseline drifts
  slightly negative). In a 24-bit signed number, the highest bit (bit 23,
  represented by the hex mask `0x800000`) is the sign bit, if it's set,
  the number is meant to be negative.
- Our `value` variable is 32 bits wide, not 24. If we left the top 8 bits
  as zero on a negative 24-bit number, it would be misread as an enormous
  *positive* number instead of the small negative number it's supposed to
  represent. `value |= 0xFF000000` fills those extra top 8 bits with 1s,
  correctly extending the negative sign across the full 32-bit width.
- **Without this:** any legitimately negative reading (which can happen,
  especially right at your zero baseline) would show up as a huge,
  nonsensical positive number instead, silently wrong data, not an error.

```c
    return value;
}
```
- Hands the final, complete, correctly-signed 24-bit measurement back to
  whatever called this function.

---

## 3. The main loop

```c
hx711_raw = HX711_Read();
HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
HAL_Delay(200);
```

- **`hx711_raw = HX711_Read();`**, calls everything above, and stores the
  result in your global variable so it's visible in Live Expressions.
- **`HAL_GPIO_TogglePin(...)`**, flips your onboard LED's state
  (on→off or off→on) every loop. This isn't functionally necessary for
  the sensor to work, it's a visual heartbeat, letting you see at a
  glance that the loop is actually running continuously rather than
  stuck. **Without it:** the code would work identically, you'd just have
  no simple visual confirmation the program is alive.
- **`HAL_Delay(200);`**, pauses execution for 200 milliseconds before the
  loop repeats. This paces how often you take a new reading (roughly 5
  times per second here, though the HX711 itself caps you at 10/sec by
  default regardless). **Without a delay:** the loop would spam readings
  as fast as the CPU allows, which isn't harmful, but makes the LED blink
  imperceptibly fast and gives you far more (and noisier) data than
  needed for simple bench testing.

---

## 4. The GPIO initialization (auto-generated by CubeMX, not hand-written)

You didn't write this by hand, CubeMX generated it from your `.ioc` pin
configuration, but it's worth understanding what it's doing:

```c
GPIO_InitTypeDef GPIO_InitStruct = {0};
GPIO_InitStruct.Pin = HX711_DT_Pin;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
GPIO_InitStruct.Pull = GPIO_NOPULL;
HAL_GPIO_Init(HX711_DT_GPIO_Port, &GPIO_InitStruct);
```

- This is a **struct** (a bundle of related settings) that configures one
  pin's behavior before you can use it. `Mode = GPIO_MODE_INPUT` tells the
  chip this pin will be *read from* (matches DT, since you're receiving
  data). A matching block elsewhere sets SCK to `GPIO_MODE_OUTPUT_PP`
  (push-pull output), since you're *driving* that pin's voltage yourself.
- **Without this initialization:** the pin would remain in its default
  reset-state configuration, which is not guaranteed to be a usable
  GPIO input/output at all, reads and writes to it would be unreliable
  or meaningless.

---

## The honest summary

Almost every bug you hit over the past few days traces back to one of
three things in this code: **timing** (reading a bit at the wrong moment
in the clock cycle), **completeness** (not waiting for a genuinely
finished measurement before reading), or **bit-width correctness** (sign
extension). That's not a coincidence, those three things are the actual
hard part of writing a bit-banged sensor protocol, and now that you've
debugged real instances of all three, you understand this pattern more
deeply than someone who just copy-pasted a working HX711 library would.
