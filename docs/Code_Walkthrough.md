# Code Walkthrough: HX711 Read Function

The custom firmware lives in `firmware/Core/Src/main.c`. Each snippet below is followed by a one-line summary.

## Global variable

```c
volatile int32_t hx711_raw = 0;
```

Holds the latest raw reading; `volatile` keeps it visible in the debugger's Live Expressions.

## Wait for data ready

```c
while (HAL_GPIO_ReadPin(HX711_DT_GPIO_Port, HX711_DT_Pin) == GPIO_PIN_SET);
```

The HX711 pulls DT low when a conversion is ready, so the loop waits until that happens.

## Clock out 24 bits

```c
for (int i = 0; i < 24; i++)
{
    HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_SET);
    value = value << 1;
    if (HAL_GPIO_ReadPin(HX711_DT_GPIO_Port, HX711_DT_Pin) == GPIO_PIN_SET)
    {
        value++;
    }
    HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_RESET);
}
```

Each clock pulse shifts in one bit, and DT is read while SCK is still high, which matches the HX711 timing.

## Set gain for the next reading

```c
HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(HX711_SCK_GPIO_Port, HX711_SCK_Pin, GPIO_PIN_RESET);
```

A 25th pulse selects channel A with gain 128 for the next conversion.

## Sign extension

```c
if (value & 0x800000)
{
    value |= 0xFF000000;
}
```

Extends the 24-bit signed result to 32 bits so negative readings stay negative.

## Main loop

```c
hx711_raw = HX711_Read();
HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
HAL_Delay(200);
```

Reads the sensor, toggles the onboard LED as a heartbeat, and waits 200 ms between readings.
