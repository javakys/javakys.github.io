---
layout: post
title: GCC기반의 Eclipse 개발 환경에서 빈 프로젝트로 W6100 EVB 프로젝트를 시작하는 방법 
date:   2019-03-31 
author: James Kim
categories: W6100
tags:	git, github, eclipse, gcc, createProject
---

## Prerequites ##
* ARM GCC Tool Chain 설치
* GCC IDE 설치
* ARM Cortex를 위한 CDT pulgin 설치

## 빈프로젝트 만들기 ##
* File->New->C/C++ Project를 선택한 후, 다음 대화상자에서 C Managed Build를 선택한다.

<img src="/assets/images/W6100_EVB/create-new-project-1.PNG" width="450" >

* "Project name"에 원하는 프로젝트 명을 입력하고 Project type은 "STM32F10x C/C++ Project"를 선택한다.

<img src="/assets/images/W6100_EVB/create-new-project-2.PNG" width="450" >

* "Chip family"에서는 "STM32f10x High Density"를 선택하고, Flash size는 256, RAM size는 48, External clock은 12000000 (12MHz)를 입력한 후, "Next"를 선택한다.

<img src="/assets/images/W6100_EVB/create-new-project-3.PNG" width="450" >

* "Next"를 클릭한다.

<img src="/assets/images/W6100_EVB/create-new-project-4.PNG" width="450" >

* "Next"를 클릭한다.

<img src="/assets/images/W6100_EVB/create-new-project-5.PNG" width="450" >

* "Tool Chain" 위치가 정확한지를 확인한 후, "Finish"를 클릭한다.

<img src="/assets/images/W6100_EVB/create-new-project-6.PNG" width="450" >

* High Density MCU는 Flash size가 크기 때문에 Flash Image Type을 "Raw Binary"로 변경한다. 좌측 "Project Exploer"에서 해당 프로젝트 이름을 선택한 후, 마우스 Right Click해서 Properties 선택하고 "C/C++ Build"->"Settings"->"Tool Setting"->"Cross ARM GNU Create Flash Image"->"General"을 선택한 후, "Output file format"에서 "Raw binary"를 선택한 후, "Apply and Close" 버튼을 클릭한다.

<img src="/assets/images/W6100_EVB/Flash-Image-type-raw-binary.PNG" width="450" >

* 프로젝트 명에서 오른쪽 마우스 클리한 후, "Biuld Project"를 선택해서 raw binary 빌드를 수행하여 빌드가 성공하는 것을 확인한다.

<img src="/assets/images/W6100_EVB/build-success.PNG" width="450" >

## Clock을 맞추기 위한 수정 ##
W6100 EVB에서 사용하는 STM32F103VC는 최대 동작 주파수가 72MHz이다. 그런데 Eclipse의 자동 생성 프로젝트는 High Density MCU의 경우에 외부 클럭 입력이 8MHze인 것을 가정해서 PLL multipler를 '9'로 고정하고 있다. 그래서 SystemClock을 확인해보면 108MHz (12*9) 인 것을 확인할 수 있다. 108MHz에서도 기본 동작에 문제가 없지만 MCU Datasheet의 권고에 맞추어서 72MHz로 지정하기 위해서 PLLmul 값을 '6'으로 변경한다.

* system->src->cmsis->system_stm32f10x.c 를 오픈한다.
* SetSysClockTo72() 함수에서 
```c
    RCC->CFGR |= (uint32_t)(RCC_CFGR_PLLSRC_HSE | RCC_CFGR_PLLMULL9);
```
를 
```c
    RCC->CFGR |= (uint32_t)(RCC_CFGR_PLLSRC_HSE | RCC_CFGR_PLLMULL6);

```
로 변경한다.

## LED 핀 수정과 동작 Test ##
Eclipse가 제공하는 기본 프로젝트는 LED를 Blinking 시키는 것이다. 프로젝트에서 지정한 LED 의 GPIO 위치가 W6100 EVB와 다르기 때문에 PIN 넘버를 수정해야한다.

include->BlinkLed.h에 LED의 Port number와 Pin number가 지정되어 있는데 , PC12 로 지정되어 있다.
```c
// Port numbers: 0=A, 1=B, 2=C, 3=D, 4=E, 5=F, 6=G, ...
#define BLINK_PORT_NUMBER               (2)
#define BLINK_PIN_NUMBER                (12)
```
반면 W6100 EVB는 RGB LED가 탑재되어 있는데 그 위치가 각각 PC6, PC8, PC9이다. 
<img src="/assets/images/W6100_EVB/RGB-LED-pin-map.png" >

이중 Blue LED인 PC9가 Blinking 하도록 Pin number를 수정한다.
```c
// Port numbers: 0=A, 1=B, 2=C, 3=D, 4=E, 5=F, 6=G, ...
#define BLINK_PORT_NUMBER               (2)
#define BLINK_PIN_NUMBER                (9)
```

파일을 저장한 후, "Build Project"를 수행하고 생성된 Binary 파일을 W6100 EVB에 다운로드 한다.

## printf문 사용환경 구축하기 ##
Embedded System에서는 개발과정에서의 디버깅 목적이나 운용중에 사용자에게 정보 전달의 목적으로 Console port로 메시지를 출력해야하는 경우가 자주 발생한다. 이를 위해서 printf()문을 사용하는 데, ARM Cortex M3 MCU에서 printf()문을 사용하기 위한 조치를 다음과 같이 수행한다.

### printf() 사용을 위한 조치 ###
gnu tool에서 printf() 함수는 내부적으로 _write() 시스템 콜을 호출한다.
따라서 _write() 함수를 수정해서 Console용 UART 포트로 문자를 출력하도록 수정해야한다.
이를 위해 이미 retarget.c를 준비하고 있다.

retarget.c에는 _write()함수를 다음과 같이 재정의해서 UART_SEND_BYTE를 호출하고 있다.
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

그리고 UART_SEND_BYTE() 함수는 어떤 UART port를 사용할 것인지에 따라서 UART peripheral을 지정해서 STM32의 USART_SendData()를 호출하도록 구성되어 있다.
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
W6100 EVB는 USART1을 Console Port를 사용한다.

이를 위해 retarget.c를 src/ 밑에 복사한다.

### USART1 을 사용하기 위한 조치 ###
ARM Cortex M3에서는 개별 Peripheral을 사용하기 위해서는 Peripheral Clock, GPIO Port 초기화, NVIC interrupt 설정 등의 조치가 필요하다.
USART1을 사용하기 위한 조치는 다음과 같다.
* USART1용 Clock Enable
* USART1 TXD, RXD가 속한 GPIO Clock Enable
* USART1 TXD, RXD용 GPIO 포트 초기화

W6100 EVB 프로젝트에서는 MCU에 의존적인 작업들은 PlatformHandler라는 디렉토리에 두고 관리하고 있다. 
1. src/ 하위디렉토리로 PlatformHandler/ 를 생성한다.
2. gpioHandler.c, gpioHandler.h, rccHandler.c, rccHandler.h, uartHandler.c, uartHandler.h 를  PlatformHandler 폴더에 생성한다.
3. rccHandler.h와 rccHandler.c 내의 코드

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

4. gpioHandler.h와 gpioHandler.c 내의 코드

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

5. uartHandler.h와 uartHandler.c 내의 코드

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

6. main() 함수에 초기화 코드 및 printf() 추가

7. include file에 대한 path 지정

8. linker option 추가

9. stm32f1-stdperiph 라이브러리 추가하기
    * Properties->C/C++ General->Paths and Symbols->"Source Location"->"/W6100-EVB-gcc-eclipse/system" 선택

    <img src="/assets/images/W6100_EVB/stm32f1x_stdperiph_library_select.PNG" width="450" >

    * "Edit Filter" 선택
    * ".../stm32f10x_usart.c"를 클릭한 후, "Remove" 클릭하고 "OK"를 선택한다.
    * "Apply and Close"를 선택한다.

10. project build후 binary를 다운로드하여 실행하면 다음과 같은 메시지 출력을 볼 수 있다.
    <img src="/assets/images/W6100_EVB/first-debug-message.PNG" width="450" >

## W6100 EVB의 주요 Peripheral 설정하기 ##
### TIMER ###
### GPIO ###
### SPI ###
### FSMC ###

## io6Library 가져오기 ##ㄴ
### git repository에 submodule로 가져오기 ###
git을 이용해서 프로젝트 버전관리를 하는 경우라면 io6Library 폴더를 최신 버전으로 업데이트 하기 위해서 submodule로 복제할 수 있다.
1. src/ 폴더로 이동한 다음, 아래 명령을 수행한다.
```shell
$ git submodule add https://github.com/Wiznet/io6Library
Cloning into 'C:/workspace/test_workspace/W6100-EVB-gcc-eclipse/src/io6Library'...
remote: Enumerating objects: 109, done.
remote: Counting objects: 100% (109/109), done.
remote: Compressing objects: 100% (81/81), done.
remote: Total 109 (delta 31), reused 74 (delta 24), pack-reused 0
Receiving objects: 100% (109/109), 2.14 MiB | 2.28 MiB/s, done.
Resolving deltas: 100% (31/31), done.
warning: LF will be replaced by CRLF in .gitmodules.
The file will have its original line endings in your working directory
```

### local에 직접 저장하는 방법 ###
1. github repository에 이동한다.
<img src="/assets/images/W6100_EVB/io6Library-github-repository.png" width="450" >
2. "Clone or download" 메뉴를 클릭한다.
<img src="/assets/images/W6100_EVB/io6Library-download-zip.png" width="450" >

## 환경 설정하기 ##
1. "Paths and Symbols" 에 라이브러리 위치 추가
<img src="/assets/images/W6100_EVB/io6Library-path-update.PNG" width="450" >

2. "Paths and Symbols"->"Source Location"에서 Exclusion pattern을 수정한다. ".../stm32f10x_tim.c", ".../stm32f10x_spi.c", ".../stm32f10x_fsmc.c" 를 리스트에서 제거한다.

3. main.c()를 수정한다.
    * 


```c

// PB_05, PB_12 pull down
*(volatile uint32_t *)(0x41003070) = 0x61; // RXDV - set pull down (PB_12)
*(volatile uint32_t *)(0x41002054) = 0x01; // PB 05 AFC
*(volatile uint32_t *)(0x41003054) = 0x61; // COL  - set pull down (PB_05)
*(volatile uint32_t *)(0x41002058) = 0x01; // PB 06 AFC
*(volatile uint32_t *)(0x41003058) = 0x61; // DUP  - set pull down (PB_06)

// PHY reset pin pull-up
*(volatile uint32_t *)(0x410020D8) = 0x01; // PD 06 AFC[00 : zero / 01 : PD06]
*(volatile uint32_t *)(0x410030D8) = 0x02; // PD 06 PADCON
*(volatile uint32_t *)(0x45000004) = 0x40; // GPIOD DATAOUT [PD06 output 1]
*(volatile uint32_t *)(0x45000010) = 0x40; // GPIOD OUTENSET    



#ifdef __DEF_USED_MDIO__ 
    
    mdio_init(GPIOB, W7500x_MDC, W7500x_MDIO); // mdio Init //
    mdio_write(GPIOB, PHYREG_CONTROL, CNTL_RESET); // PHY Reset

```

```c
while(1)
{
    val = mdio_read(GPIOB, PHYREG_STATUS);
    if(val & 0x0004) // ## debugging: PHY link
        break;
}

val=mdio_read(GPIOB, PHYREG_CONTROL); // PHY Reset
printf("PHYREG_CONTROL(0x%02x) : 0x%04x\r\n", PHYREG_CONTROL, val);
```


