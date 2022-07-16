Blank App
===

This is a blank application for the STM32F769I-DISCO board. It's based on a standard
ioc file for this board type with a minimum of changes to make it work.

Versions used:
 * STMCubeIDE 1.10.1
 * STM32Cube_FW_F7_V1.17.0
 
 How to replicate the setup
 ---
 
 Use STMCubeIDE to create a new project using 'New STM32 Project'. Then from the
 board selector select STM32F769I-DISCO.
 On the next screen make sure to select C++ if you want to use TouchGFX later on.
 Finish the generation and select yes to generate code if you get a popup about NEWLIB.
 
 I made some changes to the default IOC that made the BSP drivers compile without issues.
 * In 'System Core' deactivate WWDG
 * In 'Connectivity' set SDMMC2 to 'SD 4bits Wide bus'
 * In 'Computing' set DFSDM1 Channel 0, 1, 4 and 5 to 'Parallel input'
 * In 'Middleware' set FREERTOS/Include parameters/vTaskDelayUtil to 'Enabled'
 * In 'Middleware' set FREERTOS/Advanced settings/USE_NEWLIB_REENTRANT to 'Enabled'
 
 If you can't be bothered to deal with watchdogs immediately you can deactivate 
 the IWDG as well in 'System Core'
 
 After making these changes generate the code again 
 
 Now you have a clean project, but it's missing the board specific stuff. The
 BSP files are in supporting repository, but not automagically copied to the project.
 Locate your 'STM32Cube_FW_F7_V1.17.0' folder and copy the directory
'Drivers/BSP/STM32F769I-Discovery' to your project in the same location. 
From the 'Drivers/BSP/Components' directory copy the directories Common, adv7533,
 ft6x06, mx25l512, otm8009a and wm8994 to the same location in your project.
From the 'Utilities' Folder copy Fonts to the same location in your project.

The following directies should now show up in teh IDE after a refresh
```
./Drivers/BSP/STM32F769I-Discovery
./Drivers/BSP/Components
./Drivers/BSP/Components/ft6x06
./Drivers/BSP/Components/adv7533
./Drivers/BSP/Components/otm8009a
./Drivers/BSP/Components/Common
./Drivers/BSP/Components/wm8994
./Drivers/BSP/Components/mx25l512
./Utilities/Fonts
``` 
 
 Note: These locations will be cleaned when the ioc project is regenerated. Just copy them back at will.
 
 At this point you should be able to compile and run the project without errors. 
 It just doesn't do anything yet.
 
  
Add the following pieces of code in main.c for a quick 'Hello World'

Include the BSP
```
/* USER CODE BEGIN Includes */
#include "stm32f769i_discovery.h"
#include "stm32f769i_discovery_lcd.h"
/* USER CODE END Includes */
```

Supporting functions
```
/* USER CODE BEGIN 0 */
static uint32_t LCD_X_Size = 0;
static uint32_t LCD_Y_Size = 0;
osThreadId watchdogTaskHandle;
/**
  * @brief  On Error Handler on condition TRUE.
  * @param  condition : Can be TRUE or FALSE
  * @retval None
  */
static void OnError_Handler(uint32_t condition)
{
  if(condition)
  {
    BSP_LED_On(LED1);
    while(1) { ; } /* Blocking on error */
  }
}

void prvWatchdogTask( const void *pvParameters )
{
    portTickType        xLastWakeTime;
    (void) pvParameters;

    xLastWakeTime = xTaskGetTickCount();

    IWDG_HandleTypeDef hiwdg;
    hiwdg.Instance = IWDG;

    for( ;; )
    {
        xLastWakeTime = xTaskGetTickCount();

        HAL_IWDG_Refresh(&hiwdg);

        vTaskDelayUntil( &xLastWakeTime, pdMS_TO_TICKS(100));
    }
}

/* USER CODE END 0 */
```

Create a task to kick the watchdog
```
  /* USER CODE BEGIN RTOS_THREADS */
  osThreadDef(watchdogTask, prvWatchdogTask, osPriorityAboveNormal, 0, 1024);
  watchdogTaskHandle = osThreadCreate(osThread(watchdogTask), NULL);

  /* USER CODE END RTOS_THREADS */
```

Show stuff on the LCD and enable the green LED
```
  /* USER CODE BEGIN 2 */

  // Kick the watchdog until the task is running
  HAL_IWDG_Refresh(&hiwdg);

  uint8_t  lcd_status = LCD_OK;
  lcd_status = BSP_LCD_Init();
  OnError_Handler(lcd_status != LCD_OK);

  BSP_LCD_LayerDefaultInit(0, LCD_FB_START_ADDRESS);
  BSP_LCD_SelectLayer(0);

  // Kick the watchdog until the task is running
  HAL_IWDG_Refresh(&hiwdg);

  /* Get the LCD Width and Height*/
  LCD_X_Size = BSP_LCD_GetXSize();
  LCD_Y_Size = BSP_LCD_GetYSize();

  BSP_LCD_Clear(LCD_COLOR_BLUE);
  BSP_LCD_SetBackColor(LCD_COLOR_BLUE);
  BSP_LCD_SetTextColor(LCD_COLOR_BLUE);
  BSP_LCD_FillRect(0, 0, LCD_X_Size, 112);

  BSP_LCD_SetTextColor(LCD_COLOR_WHITE);
  BSP_LCD_DisplayStringAt(0,LINE(2) , (uint8_t *)"Hello World", CENTER_MODE);
  BSP_LCD_SetFont(&Font16);
  BSP_LCD_DisplayStringAt(0,LINE(5), (uint8_t *)"Hello World example with DSI LCD", CENTER_MODE);
  BSP_LCD_DisplayStringAt(0,LINE(6), (uint8_t *)"made for the F769I Discovery board", CENTER_MODE);

  BSP_LED_Init(LED2);
  BSP_LED_Toggle(LED2);

  /* USER CODE END 2 */
```

Enable the red led on HAL init error.
```
  /* USER CODE BEGIN Error_Handler_Debug */
  /* User can add his own implementation to report the HAL error return state */
  __disable_irq();
  BSP_LED_Init(LED1);
  BSP_LED_On(LED1);
  while (1)
  {
  }
  /* USER CODE END Error_Handler_Debug */
```

Note: The program will fail if there is no SD card in the slot. Comment out the line with `MX_SDMMC2_SD_Init();` to avoid this.
