# MicromouseDocs
Useful (maybe) Documentation to program the micromouse in C. All that I am going to talk about is mainly how to use each component individually, nothing more.

>**NOTE:** "By using C, you may be shooting yourself in the foot, and as such I take no responsibility for any failure whatsoever."

Feel free to fork this document and make changes if you find something useful. This is the power of open source, utilize it and better your fellow human.

**All the code comes from the google drive that they gave us, so please don't DM me asking for where I got the code**

---

## Fixing the Makefile
I can't really help you other giving you my current Makefile, although you may need to change some things since this is the Makefile I am currently using to test each component so depending on what part I am testing certain things may be commented out.

**NOTE: Windows Users will have to change the path for the includes (should only be for the /usr/include), but since I don't use Windows I have no clue what to change it to**

## Common fixes
- Turn the board on
- Try pressing Reset
- Flash it again after recompiling
- IDK, it's electronics

## Flashing the board
> You probably need the STM32cube programmer (tried using st-flash [st-flash is a terminal utility to flash binaries to stm32 boards] but it wasn't overwriting the old code so eventually ended up running out of space on the mouse)

Basically just follow the steps you used to flash the inital program (connect embedded debugger, click connect, find the .elf file, click download, finally disconnect and test)
###To find the compiled .elf file (dunno if you can use the other types e.g .BIN , never tried nor do I care to)

Navigate to the root of the project, then look inside build. If you compilied everything correctly then you should see a MicroMouseProgramming.elf, that's the new program that you flash.

---

## Testing Each Component:
Once you can compile your code, let's begin with testing each piece of code individually. To make this alot easier I suggest setting up an interrupt using SW1 and SW2. The other switch is hard wired to the reset of the mircocontroller so you can't modify it's function.

To set up the interrrupt for SW1, add the following code to stm32l4xx_it.c:
```
void
EXTI9_5_IRQHandler (void)
{
  if (LL_EXTI_IsActiveFlag_0_31 (LL_EXTI_LINE_6) != RESET)
    {
      // Clear the interrupt flag
      LL_EXTI_ClearFlag_0_31 (LL_EXTI_LINE_6);
      // Whatever you want to run
    }
}

void
EXTI2_IRQHandler (void)
{
  if (LL_EXTI_IsActiveFlag_0_31 (LL_EXTI_LINE_2) != RESET)
    {
      LL_EXTI_ClearFlag_0_31 (LL_EXTI_LINE_2);
      // Whatever you want to run
    }
}
```
 
 You should also probably add the function prototypes to the corresponding .h file ( in this case stm32l4xx_it.h)
 
```
void EXTI9_5_IRQHandler(void);
void EXTI2_IRQHandler(void);
```


Don't forget to initialize the interrupts in the gpio.c

```
  // Enable the EXTI line for the SW1 pin
  LL_SYSCFG_SetEXTISource(LL_SYSCFG_EXTI_PORTE, LL_SYSCFG_EXTI_LINE6);
  LL_EXTI_EnableIT_0_31(LL_EXTI_LINE_6);
  LL_EXTI_EnableRisingTrig_0_31(LL_EXTI_LINE_6);

  // Configure the NVIC for the EXTI line
  NVIC_SetPriority(EXTI9_5_IRQn, 0);
  NVIC_EnableIRQ(EXTI9_5_IRQn);

  // Configure EXTI line for SW2_Pin
  LL_SYSCFG_SetEXTISource(LL_SYSCFG_EXTI_PORTB, LL_SYSCFG_EXTI_LINE2);
  LL_EXTI_EnableIT_0_31(LL_EXTI_LINE_2);
  LL_EXTI_EnableRisingTrig_0_31(LL_EXTI_LINE_2);

  // Enable EXTI interrupt in NVIC
  NVIC_SetPriority(EXTI2_IRQn, 0);
  NVIC_EnableIRQ(EXTI2_IRQn);

```

Now that you have the interrupt to test stuff (I mainly used it for the motors [you know, to prevent my mouse from shooting off into the nearest wall] but also for the LEDs since it was already setup).

> You might want to modify the main function to only run your code, so comment out all other functions that donn't actually matter for the component you testing.

Here is an example main function that I'm using to test the Gyro and Accelerometer:

```
  HAL_Init ();
  SystemClock_Config ();
  PeriphCommonClock_Config ();
  MX_GPIO_Init ();
  /*MX_DMA_Init();*/
  /*MX_ADC1_Init();*/
  /*MX_USB_DEVICE_Init();*/
  /*MX_TIM3_Init();*/
  MX_I2C2_Init ();
  /*MX_ADC2_Init();*/

  initIMU ();

  /*testingLoops ();*/

  while (1)
    {
      refreshIMUValues ();
      // CustomWhile();
    }


```

### Motors
The motors are turned on by firstly enabling the enable pin, then setting the output high on the forward pin of the left or right depending on which one you want to spin

> I'm not sure if you're meant to drive it this way (would make more sense to use PWM but I can't be bothered to add the addtional documentation) since it's quite loud and might not be good for the gears but this is a quick and dirty test so just don't leave the motor running for a long time

The GPIO needs to be configured, you can do it with something like this:

```
  /* Custom Init Section Used to enable motors, copied from the LED output -SB */
  GPIO_InitStruct.Pin = MOT_RIGHT_BWD_Pin | MOT_RIGHT_FWD_Pin |
                        MOT_LEFT_BWD_Pin | MOT_LEFT_FWD_Pin;
  GPIO_InitStruct.Mode = LL_GPIO_MODE_OUTPUT;
  GPIO_InitStruct.Speed = LL_GPIO_SPEED_FREQ_LOW;
  GPIO_InitStruct.OutputType = LL_GPIO_OUTPUT_PUSHPULL;
  GPIO_InitStruct.Pull = LL_GPIO_PULL_NO;
  LL_GPIO_Init(GPIOC, &GPIO_InitStruct);
```

Once it's configured it's quite simple, just set the enable pin high then enable whatever pin you want corresponding to the direction and motor you want.

>The schematic is incorrect when it says there are 2 enable pins, there is a single enable pin and it's PIN D7 (if I'm not mistaken, just check the main.h to be sure). Pin C7 should be the right motor's forward or back depending on how you soldered it. Pins C8 and C9 are the left motor's control so figure out which one does what. **Don't forget that the front of the mouse is where the sensor board is**

Setting the output high is basically just:

```
LL_GPIO_SetOutputPin (MOTOR_EN_GPIO_Port, MOTOR_EN_Pin);

// Move the Right motor forward
LL_GPIO_SetOutputPin (MOT_RIGHT_FWD_GPIO_Port, MOT_RIGHT_FWD_Pin);
LL_GPIO_ResetOutputPin (MOT_RIGHT_BWD_GPIO_Port, MOT_RIGHT_BWD_Pin);

// Move the Left motor forward
LL_GPIO_SetOutputPin (MOT_LEFT_FWD_GPIO_Port, MOT_LEFT_FWD_Pin);
LL_GPIO_ResetOutputPin (MOT_LEFT_BWD_GPIO_Port, MOT_LEFT_BWD_Pin);

```

To turn it off, just reset the pins:

```
// Disable the Motors
LL_GPIO_ResetOutputPin (MOT_RIGHT_FWD_GPIO_Port, MOT_RIGHT_FWD_Pin);
LL_GPIO_ResetOutputPin (MOT_RIGHT_BWD_GPIO_Port, MOT_RIGHT_BWD_Pin);

LL_GPIO_ResetOutputPin (MOT_LEFT_FWD_GPIO_Port, MOT_LEFT_FWD_Pin);
LL_GPIO_ResetOutputPin (MOT_LEFT_BWD_GPIO_Port, MOT_LEFT_BWD_Pin);

LL_GPIO_ResetOutputPin (MOTOR_EN_GPIO_Port, MOTOR_EN_Pin);

```

Just add the above to the interrupts and you should be able to start the motors by pressing a switch and turning it off by pressing the other switch.
> If your motors don't start maybe try pressing reset

### Onboard LEDs (D4, D5, D6)
Same procedure as motors, set the enable pin (spent way too much time figuring out that this has an enable pin) then set the pin, corresponding to the LED that you want to illuminate, high.
```
// Enable the LEDs
LL_GPIO_SetOutputPin (CTRL_LEDS_GPIO_Port, CTRL_LEDS_Pin);

LL_GPIO_SetOutputPin (LED0_GPIO_Port, LED0_Pin);

```

The GPIO pins should already be configured but if not just copy the motor's GPIO and edit the ports and pins (take a look at the main.h file)

Turning it off is the same as with the motors, pull the pins low:
```
// Disable the LEDs
LL_GPIO_ResetOutputPin (LED0_GPIO_Port, LED0_Pin);

//Disable the enable pin
LL_GPIO_ResetOutputPin (CTRL_LEDS_GPIO_Port, CTRL_LEDS_Pin);

```

### IR LEDs
Just for clarification the clear LEDs should be the IR emitters.

This time no enable pin so quite easy (Check the main.h for the different sides and directions):
```
      // Start the IR LEDs on the Left Side facing down and Right Side facing down
      LL_GPIO_SetOutputPin (LED_DOWN_LS_GPIO_Port, LED_DOWN_LS_Pin);
      LL_GPIO_SetOutputPin (LED_DOWN_RS_GPIO_Port, LED_DOWN_RS_Pin);

      // Stop the IR LEDs
      LL_GPIO_ResetOutputPin (LED_DOWN_LS_GPIO_Port, LED_DOWN_LS_Pin);
      LL_GPIO_ResetOutputPin (LED_DOWN_RS_GPIO_Port, LED_DOWN_RS_Pin);

```

### Fallback to enable everything above:

```
void
EXTI9_5_IRQHandler (void)
{
  if (LL_EXTI_IsActiveFlag_0_31 (LL_EXTI_LINE_6) != RESET)
    {
      // Clear the interrupt flag
      LL_EXTI_ClearFlag_0_31 (LL_EXTI_LINE_6);
      // Enable the Motors
      LL_GPIO_SetOutputPin (MOTOR_EN_GPIO_Port, MOTOR_EN_Pin);

      // Enable the LEDs
      LL_GPIO_SetOutputPin (CTRL_LEDS_GPIO_Port, CTRL_LEDS_Pin);

      LL_GPIO_SetOutputPin (LED0_GPIO_Port, LED0_Pin);
      LL_GPIO_SetOutputPin (LED1_GPIO_Port, LED1_Pin);

      // Start the IR LEDs
      LL_GPIO_SetOutputPin (LED_DOWN_LS_GPIO_Port, LED_DOWN_LS_Pin);
      LL_GPIO_SetOutputPin (LED_DOWN_RS_GPIO_Port, LED_DOWN_RS_Pin);

      // Move the Right motor forward
      LL_GPIO_SetOutputPin (MOT_RIGHT_FWD_GPIO_Port, MOT_RIGHT_FWD_Pin);
      LL_GPIO_ResetOutputPin (MOT_RIGHT_BWD_GPIO_Port, MOT_RIGHT_BWD_Pin);

      // Move the Left motor forward
      LL_GPIO_SetOutputPin (MOT_LEFT_FWD_GPIO_Port, MOT_LEFT_FWD_Pin);
      LL_GPIO_ResetOutputPin (MOT_LEFT_BWD_GPIO_Port, MOT_LEFT_BWD_Pin);
    }
}

void
EXTI2_IRQHandler (void)
{
  if (LL_EXTI_IsActiveFlag_0_31 (LL_EXTI_LINE_2) != RESET)
    {
      LL_EXTI_ClearFlag_0_31 (LL_EXTI_LINE_2);

      // Disable the Motors
      LL_GPIO_ResetOutputPin (MOT_RIGHT_FWD_GPIO_Port, MOT_RIGHT_FWD_Pin);
      LL_GPIO_ResetOutputPin (MOT_RIGHT_BWD_GPIO_Port, MOT_RIGHT_BWD_Pin);

      LL_GPIO_ResetOutputPin (MOT_LEFT_FWD_GPIO_Port, MOT_LEFT_FWD_Pin);
      LL_GPIO_ResetOutputPin (MOT_LEFT_BWD_GPIO_Port, MOT_LEFT_BWD_Pin);

      LL_GPIO_ResetOutputPin (MOTOR_EN_GPIO_Port, MOTOR_EN_Pin);

      // Disable the LEDs
      LL_GPIO_ResetOutputPin (LED0_GPIO_Port, LED0_Pin);
      LL_GPIO_ResetOutputPin (LED1_GPIO_Port, LED1_Pin);

      LL_GPIO_ResetOutputPin (CTRL_LEDS_GPIO_Port, CTRL_LEDS_Pin);

      // Stop the IR LEDs
      LL_GPIO_ResetOutputPin (LED_DOWN_LS_GPIO_Port, LED_DOWN_LS_Pin);
      LL_GPIO_ResetOutputPin (LED_DOWN_RS_GPIO_Port, LED_DOWN_RS_Pin);
    }
}
```

### ADC
TODO

### Gyro and Accelerometer
TODO




