---
layout: post
title: How to make a new W6100 project with gcc and eclipse 
date:   2019-03-25 
author: James Kim
categories: W6100
---

## Prerequites ##
* Installation of ARM GCC Tool Chain
* Installation of GCC IDE
* Installation of CDT plugin for ARM Cortex

## Creating an emply project ##
* Select File->New->C/C++ Project, then choose "C Managed Build" on the next dialogbox.

<img src="/assets/images/W6100_EVB/create-new-project-1.PNG" width="450" >

* Input your project name on "Project name" field and select "STM32F10x C/C++ Project" in Project type.

<img src="/assets/images/W6100_EVB/create-new-project-2.PNG" width="450" >

* Select "STM32f10x High Density" from "Chip family", write 256 in Flash size, 48 in RAM and 12000000 (12MHz) in External clock, and then click "Next".

<img src="/assets/images/W6100_EVB/create-new-project-3.PNG" width="450" >

* Click "Next".

<img src="/assets/images/W6100_EVB/create-new-project-4.PNG" width="450" >

* Click "Next".

<img src="/assets/images/W6100_EVB/create-new-project-5.PNG" width="450" >

* Check the location of "Tool Chain" and click "Finish".

<img src="/assets/images/W6100_EVB/create-new-project-6.PNG" width="450" >

* Because the Flash size of High Density MCU is beyond the limit of HEX file format, change "Flash Image Type" to "Raw Binary". Choose the project name from "Project Exploer" on the left side of the tool, then click the right button of the mouse and select "Properties" from the menu list. Then, select "C/C++ Build"->"Settings"->"Tool Setting"->"Cross ARM GNU Create Flash Image"->"General" and change "Output file format" to "Raw binary", then click "Apply and Close" button.

<img src="/assets/images/W6100_EVB/Flash-Image-type-raw-binary.PNG" width="450" >

* Click the right button of the mouse with the selected project name, then choose  "Biuld Project" and check if the build succeeds.

<img src="/assets/images/W6100_EVB/build-success.PNG" width="450" >

## Changes for Clock adjustment ##
STM32F103VC on W6100 EVB supports the maximum clock frequency 72MHz. However, the project which is created automatically by Eclipse is set with PLL multipler '9' for High Density MCU because it assumes the external clock as 8MHz. So you can find SystemClock is 108MHz (12*9). Fortunately the MCU works wel with 108MHz but the clock beyond 72MHz is not recommended so that PLLmul should be changed to '6'.

* Open system->src->cmsis->system_stm32f10x.c.
* in the function SetSysClockTo72(), edit from

```c
    RCC->CFGR |= (uint32_t)(RCC_CFGR_PLLSRC_HSE | RCC_CFGR_PLLMULL9);
```
to

```c
    RCC->CFGR |= (uint32_t)(RCC_CFGR_PLLSRC_HSE | RCC_CFGR_PLLMULL6);

```


## Edit the setting for LED pin and test the operation ##
The default project is blinking a LED. We need to change the PIN number of LED because of difference of LED pin definition between the default project and W6100 EVB.

LED Port number and Pin number are defined as PC12 in include->BlinkLed.h.

```c
// Port numbers: 0=A, 1=B, 2=C, 3=D, 4=E, 5=F, 6=G, ...
#define BLINK_PORT_NUMBER               (2)
#define BLINK_PIN_NUMBER                (12)
```
Whereas, RGB LEDs on W6100 EVB is connected to PC6, PC8 and PC9 respectively. 
<img src="/assets/images/W6100_EVB/RGB-LED-pin-map.png" >

We'll change the PIN number to PC9 for using Blue LED..
```c
// Port numbers: 0=A, 1=B, 2=C, 3=D, 4=E, 5=F, 6=G, ...
#define BLINK_PORT_NUMBER               (2)
#define BLINK_PIN_NUMBER                (9)
```

Save the file and do "Build Project", then download the binary file onto W6100 EVB.

## Setting up for using printf statement ##
In Embedded Systems, outputting messages on console port is required so often for debugging in development or showing information to users in running. We use printf() statement for this purpose, we need below to make printf() available with ARM Cortex M3 MCU. .

### Process for using printf() ###
In GNU tool, printf() calls _write() function internally.
So, we will update _write() to send a byte to UART port for console.
retarget.c is the file for this.

In retarget.c, _write() is updated to call UART_SEND_BYTE like below.
```c
__attribute__ ((used))  int _write (int fd, char *ptr, int len)
{
  size_t i;
  for (i=0; i<len;i++) {
    UART_SEND_BYTE(ptr[i]); // call character output function
    }
  return len;
}
```

And UART_SEND_BYTE() set UART peripheral depending on which UART port will be used, and call USART_SendData().
```c
// Replaced the defines for UART Selector to Callback functions.
#define USING_UART1
#if defined (USING_UART0)
	#define UART_SEND_BYTE(ch)  UartPutc(USART0,ch)
	#define UART_RECV_BYTE()    UartGetc(USART0)
#elif defined (USING_UART1)
	#define UART_SEND_BYTE(ch)  UartPutc(USART1,ch)
	#define UART_RECV_BYTE()    UartGetc(USART1)
#elif defined (USING_UART2)
	#define UART_SEND_BYTE(ch)  UartPutc(USART2,ch)
	#define UART_RECV_BYTE()    UartGetc(USART2)
#endif


uint8_t UartPutc(USART_TypeDef* UARTx, uint8_t ch)
{
    USART_SendData(UARTx, (uint16_t)ch);

    while(USART_GetFlagStatus(UARTx, USART_FLAG_TXE) == RESET);

    return (ch);
}
```
On W6100 EVB, USART1 is set for Console Port.

Please copy retarget.c to src/ directory.

### Process of enabling USART1 ###
To use each peripheral in ARM Cortex M3, we need setting of Peripheral Clock, initializing GPIO Port and setting of NVIC interrupt.
Please do below using USART1.
* Make USART1 Clock enable
* Make GPIO Clock for USART1 TXD and RXD enable
* Initialzie GPIO port for USART1 TXD and RXD

W6100 EVB project prepares the functions, which is dependent to the MCU, in PlatformHandler folder. 
1. Make PlatformHandler/ under src/
2. Make gpioHandler.c, gpioHandler.h, rccHandler.c, rccHandler.h, uartHandler.c, uartHandler.h in PlatformHandler folder
3. Edit rccHandler.h and rccHandler.c with below codes

rccHandler.h
```h 
#ifndef PLATFORMHANDLER_RCCHANDLER_H_
#define PLATFORMHANDLER_RCCHANDLER_H_


#include "stm32f10x_rcc.h"

void RCC_Configuration(void);

#endif /* PLATFORMHANDLER_RCCHANDLER_H_ */
```

rccHandler.c
```c
#include "rccHandler.h"


void RCC_Configuration(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA , ENABLE );
}
```

4. Edit gpioHandler.h and gpioHandler.c with below codes

gpioHandler.h
```h
#ifndef PLATFORMHANDLER_GPIOHANDLER_H_
#define PLATFORMHANDLER_GPIOHANDLER_H_


#include "stm32f10x_gpio.h"

void GPIO_Configuration(void);

#endif /* PLATFORMHANDLER_GPIOHANDLER_H_ */

```

gpioHandler.c
```c
#include "gpioHandler.h"

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
}

```

5. Edit uartHandler.h and uartHandler.c with below codes

uartHandler.h
```h
#ifndef PLATFORMHANDLER_UARTHANDLER_H_
#define PLATFORMHANDLER_UARTHANDLER_H_


#include "stm32f10x_usart.h"

#define DEBUG_UART USART1

void USART_Configuration(void);

#endif /* PLATFORMHANDLER_UARTHANDLER_H_ */
```

uartHandler.c
```c
#include "uartHandler.h"

void USART_Configuration(void)
{
	USART_InitTypeDef USART_InitStructure;

	USART_InitStructure.USART_BaudRate = 115200;
	USART_InitStructure.USART_WordLength = USART_WordLength_8b;
	USART_InitStructure.USART_StopBits = USART_StopBits_1;
	USART_InitStructure.USART_Parity = USART_Parity_No;
	USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
	USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;

	USART_Init(DEBUG_UART, &USART_InitStructure);
	USART_Cmd(DEBUG_UART, ENABLE);
}
```

6. Add the initialization code and printf() statement in main()

7. Set the path for "include files"

8. Add linker option

9. Add stm32f1-stdperiph library
    * Select Properties->C/C++ General->Paths and Symbols->"Source Location"->"/W6100-EVB-gcc-eclipse/system"

    <img src="/assets/images/W6100_EVB/stm32f1x_stdperiph_library_select.PNG" width="450" >

    * Select "Edit Filter"
    * Click ".../stm32f10x_usart.c", then choose "Remove" and click "OK".
    * Select "Apply and Close".

10. Build project and download the binary, then you can see the message like in below image.
    <img src="/assets/images/W6100_EVB/first-debug-message.PNG" width="450" >


