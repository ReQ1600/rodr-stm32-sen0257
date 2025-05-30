# STM32 SEN0257 MODULE
Module for reading data from sen0257 pressure sensor and filtering it.

Tested on **Nucleo-F767ZI**

## Requirements
 - HAL
 - UART

## Example for FreeRTOS with CMSIS v1
```c
void startGetPressureTask(void const * argument)
{
  /* USER CODE BEGIN startGetPressureTask */
	float voltage = 0;
	float medianV = 0;

	float voltage_samples[SHORT_WIN];
	int short_id = 0;
	bool short_filled = false;

	float long_term_filtered[LONG_WIN];
	int long_id = 0;
	bool long_filled = false;

	float fir_avg_buff[LONG_WIN] = { 0 };
	int fir_avg_id= 0;
	bool fir_avg_filled = false;

	float long_fir_avg = 0.0f;
	float long_fir = 0.0f;

	float pressure = 0;
  /* Infinite loop */
	for(;;)
	{
		voltage = SEN0257_readStableAnalog(5, &hadc1) * 5.0f / 4095.0f;

		voltage_samples[short_id] = voltage;
		short_id = GET_AVG_ID(short_id, SHORT_WIN);

		if (short_id == 0) short_filled = true;


		if (short_filled)
		{
			medianV = SEN0257_getMedian(voltage_samples, SHORT_WIN);

			long_term_filtered[long_id] = medianV;
			long_id = GET_AVG_ID(long_id, LONG_WIN);

			if (long_id == 0) long_filled = true;
		}

		fir_avg_buff[fir_avg_id] = medianV;
		fir_avg_id = GET_AVG_ID(fir_avg_id, LONG_WIN);

		if (fir_avg_id == 0) fir_avg_filled = true;

		//calculate only when buffer is full
		if (fir_avg_filled)
		{
			float sum = 0.0f;
			for (int i = 0; i < LONG_WIN; i++)
				sum += fir_avg_buff[i];

			long_fir_avg = sum / LONG_WIN;
		}
		else
			long_fir_avg = medianV;  // use current val until buff filled

		pressure = (long_fir_avg - OFFSET) * 250.0f;//this is output so it should be queued

		//packing float into int and putting it into queue
		osMessagePut(pressureQueueHandle, *(int32_t *)&pressure, osWaitForever);

		osDelay(10);//should be changed to fit whatever you want to do
	}
  /* USER CODE END startGetPressureTask */
}
```
