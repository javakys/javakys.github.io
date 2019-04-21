---
layout: post
title: How to set the project up for W6100 EVB 
date:   2019-03-26
author: James Kim
categories: W6100
---

## Hardware Configuration ##
<img src="/assets/images/W6100_EVB/W6100-EVB-MCU-Pin-map.png">

* FSMC
    * FSMC_NADV - PB7
    * FSMC_NOE - PD4
    * FSMC_NWE - PD5
    * DA0 - PD14
    * DA1 - PD15
    * DA2 - PD0
    * DA3 - PD1
    * DA4 - PE7
    * DA5 - PE8
    * DA6 - PE9
    * DA7 - PE10
* SPI
    * SPI2_SCK - PB13
    * SPI2_MISO - PB14
    * SPI2_MOSI - PB15
    * SPI2_NSS - PD7
* USART
    * USART1_TX - PA9
    * USART1_RX - PA10
* INT
    * RSTn - PD8
    * INTn - PD9     
## RCC Configuration ##

```c
void RCC_Configuration(void)
{

	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2 | RCC_APB1Periph_SPI2, ENABLE);

	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA |                              RCC_APB2Periph_GPIOB | RCC_APB2Periph_GPIOD |                               RCC_APB2Periph_GPIOE, ENABLE );

	RCC_AHBPeriphClockCmd(RCC_AHBPeriph_FSMC, ENABLE);

}
```
## NVIC Configuration ##

```c

```
## GPIO Configuration ##

```c
void GPIO_Configuration(void)
{
	GPIO_InitTypeDef GPIO_InitStructure;

	// Configure pin in output push/pull mode
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9 ;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);

	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10 ;
	GPIO_Init(GPIOA, &GPIO_InitStructure);

	/*For SPI*/
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = SPI2_SCK_PIN | SPI2_MOSI_PIN | SPI2_MISO_PIN;
	GPIO_Init(SPI2_SCK_PORT, &GPIO_InitStructure);

	/*For CS*/
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitStructure.GPIO_Pin = SPI2_NSS_PIN;
	GPIO_Init(SPI2_NSS_PORT, &GPIO_InitStructure);

	/*For Reset*/
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitStructure.GPIO_Pin = RSTn_PIN;
	GPIO_Init(RSTn_PORT, &GPIO_InitStructure);

	/* GPIOD configuration for data 0~3 and */
	GPIO_InitStructure.GPIO_Pin = DA0_PIN | DA1_PIN | DA2_PIN | DA3_PIN;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

	GPIO_Init(DA0_PORT, &GPIO_InitStructure);

	/* GPIOE configuration for fsmc data d4~d7*/
	GPIO_InitStructure.GPIO_Pin = DA4_PIN | DA5_PIN  | DA6_PIN  | DA7_PIN ;
	GPIO_Init(DA4_PORT, &GPIO_InitStructure);

	/* GPIOB configuration for fsmc nadv */
	GPIO_InitStructure.GPIO_Pin = FSMC_NADV_PIN;
	GPIO_Init(FSMC_NADV_PORT, &GPIO_InitStructure);

	/* NOE and NWE configuration */
	GPIO_InitStructure.GPIO_Pin = FSMC_NOE_PIN | FSMC_NWE_PIN;
	GPIO_Init(FSMC_NOE_PORT, &GPIO_InitStructure);

}
```

## FSMC Configuration ##

```c
void FSMC_Configuration(void)
{
	FSMC_NORSRAMInitTypeDef  FSMC_NORSRAMInitStructure;
	FSMC_NORSRAMTimingInitTypeDef  p;

	FSMC_NORSRAMCmd(FSMC_Bank1_NORSRAM1, DISABLE);

	p.FSMC_AddressSetupTime = 0x03;
	p.FSMC_AddressHoldTime = 0x01;
	p.FSMC_DataSetupTime = 0x08;
	p.FSMC_BusTurnAroundDuration = 0;
	p.FSMC_CLKDivision = 0x00;
	p.FSMC_DataLatency = 0;
	p.FSMC_AccessMode = FSMC_AccessMode_B;


	FSMC_NORSRAMInitStructure.FSMC_Bank = FSMC_Bank1_NORSRAM1;
	FSMC_NORSRAMInitStructure.FSMC_DataAddressMux = FSMC_DataAddressMux_Enable;
	FSMC_NORSRAMInitStructure.FSMC_MemoryType = FSMC_MemoryType_NOR;
	FSMC_NORSRAMInitStructure.FSMC_MemoryDataWidth = FSMC_MemoryDataWidth_8b;
	FSMC_NORSRAMInitStructure.FSMC_BurstAccessMode = FSMC_BurstAccessMode_Disable;
	FSMC_NORSRAMInitStructure.FSMC_WaitSignalPolarity = FSMC_WaitSignalPolarity_Low;
	FSMC_NORSRAMInitStructure.FSMC_WrapMode = FSMC_WrapMode_Disable;
	FSMC_NORSRAMInitStructure.FSMC_WaitSignalActive = FSMC_WaitSignalActive_BeforeWaitState;
	FSMC_NORSRAMInitStructure.FSMC_WriteOperation = FSMC_WriteOperation_Enable;
	FSMC_NORSRAMInitStructure.FSMC_WaitSignal = FSMC_WaitSignal_Disable;
	FSMC_NORSRAMInitStructure.FSMC_ExtendedMode = FSMC_ExtendedMode_Disable;
	FSMC_NORSRAMInitStructure.FSMC_WriteBurst = FSMC_WriteBurst_Disable;
	FSMC_NORSRAMInitStructure.FSMC_ReadWriteTimingStruct = &p;
	FSMC_NORSRAMInitStructure.FSMC_WriteTimingStruct = &p;
	FSMC_NORSRAMInitStructure.FSMC_AsynchronousWait = FSMC_AsynchronousWait_Disable;

	FSMC_NORSRAMInit(&FSMC_NORSRAMInitStructure);

	/*!< Enable FSMC Bank1_SRAM1 Bank */
	FSMC_NORSRAMCmd(FSMC_Bank1_NORSRAM1, ENABLE);


}
```

## SPI Configuration ##

```c
void SPI_Configuration(void)
{
	SPI_InitTypeDef SPI_InitStructure;
	SPI_InitStructure.SPI_Direction = SPI_Direction_2Lines_FullDuplex;
	SPI_InitStructure.SPI_DataSize = SPI_DataSize_8b;
	SPI_InitStructure.SPI_CPOL = SPI_CPOL_High;
	SPI_InitStructure.SPI_CPHA = SPI_CPHA_2Edge;
	SPI_InitStructure.SPI_NSS = SPI_NSS_Soft;
	SPI_InitStructure.SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_2;//SPI_BaudRatePrescaler_4;
	SPI_InitStructure.SPI_FirstBit = SPI_FirstBit_MSB;
	SPI_InitStructure.SPI_CRCPolynomial = 7;
	/* Initializes the SPI communication */
	SPI_InitStructure.SPI_Mode = SPI_Mode_Master;

	SPI_Init(SPI2, &SPI_InitStructure);

	SPI_Cmd(SPI2,ENABLE);

}
```
## Build and Download ##