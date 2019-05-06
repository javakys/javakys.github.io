---
layout: post
title: STM32F103과 W6100을 이용한 심플 TFTP Client 프로그램 구현 
date:   2019-05-06
author: James Kim
categories: W6100
---

## 준비물 ##
* W6100 EVB

  * STM32F103VC와 W6100으로 구성된 Evaluation Board 
* ARM GCC Tool Chain 
* Eclipse IDE

## TFTP Client의 동작 시나리오 ##
TFTP(Trivial File Transfer Protocol) 는 이름에서 알 수 있는 것 처럼 단순화된 방식으로 파일을 전송하는 프로토콜이다.
아주 작은 메모리만을 가진 시스템에서도 TFTP를 구현할 수 있기 때문에 소규모 장치의 부팅 코드를 다운로드하기 위한 용도로 사용되었다.
요즘에도 부트 코드만 탑재된 장치에서 어플리케이션 코드를 다운로드 받기위한 목적으로 많이 사용하고 있다. 그렇기 때문에 TFTP에 대해서 잘 이해하고 있다면 활용도가 높을 것이다.

이 예제에서는 파일 시스템을 갖추고 있지 않기 때문에 리눅스나 윈도우즈 같은 OS 기반의 시스템에서의 TFTP보다는 훨씬 단순한 방식으로 파일 송수신 과정을 확인한다.

### TFTP 프로토콜 흐름도 및 데이터 포맷 ###
1. TFTP 프로토콜 흐름도는 아래 그림과 같다.

![_config.yml](/assets/images/2019-5-6/TFTP-data-flow.png)

* TFTP Client가 File을 수신하기 위한 Read Request(RRQ) 또는 File을 송신하기 위한 Write Request(WRQ) 명령을 TFTP Server에 전송함에 의해 TFTP 절차가 개시된다.
* RRQ(또는 WRQ)에는 Option이 포함될 수 있는 데, Option이 포함되었는지 여부에 따라서 Peer 시스템으로부터의 응답이 달라진다.
* Option이 없는 경우에는 'DATA' 패킷을 받게 되고, Option이 있는 경우에는 'OACK' 패킷을 받게 된다.
* 요청한 파일이 있는 경우에는 서버는 'DATA' 패킷을 전송한다. 이때, 패킷이 몇번째 Block 인지를 지정하는 Block ID가 붙어오게된다.
* 'DATA' 패킷을 수신하면 수신한 Block ID를 가지고 'ACK' 패킷을 전송한다.
* 'DATA' 패킷은 지정된 Block size 크기의 data를 가지고 전송하는데, Block size보다 작은 크기의 data 패킷이 전송되면 이것이 '마지막' 패킷이라는 의미이다. 전송할 파일 사이즈가 Block size의 정수배인 경우라도 마지막에 data size가 '0'인 'DATA' 패킷이 반드시 전송되어져야 한다.
* 마지막 'DATA' 패킷에 대한 'ACK'를 전송함으로써 TFTP 통신은 종료된다.

2. 데이터 포맷
TFTP 통신에서 주고 받는 명령 또는 응답의 메시지 구조는 아래 그림과 같다.
![_config.yml](/assets/images/2019-5-6/message_format.PNG)
* 길이가 지정되어 있는 경우를 제외하고는 각 필드의 뒤에는 'NULL' 문자가 위치함으로써 해당 필드가 끝났다는 것을 표시한다.

## Simple TFTP Client의 코드 구성 ##
1. UDP 소켓 생성
```c
    case SOCK_CLOSED:
        socketcreate(sn, TFTP_TEMP_PORT, ip_mode);
        printf("curr state: STATE_NONE\r\n");
printf("%d:Opened, UDP loopback, port [%d] as %s\r\n", sn, TFTP_TEMP_PORT, get_mode_message(ip_mode));
```

2. RRQ 패킷 전송으로 TFTP 세션 개시
```c
    case SOCK_UDP:
        switch(g_tftp_state)
        {
        case STATE_NONE:

            ret = send_rrq(buf, filename, sn, server_ip, ip_mode);

            if(ret > 0){
                printf("curr state: STATE_RRQ\r\n");
                g_tftp_state = STATE_RRQ;
            }

break;

...

uint16_t send_rrq(uint8_t * buf, uint8_t* filename, uint8_t sn, uint8_t* server_ip, uint8_t ip_mode)
{
    uint16_t buf_size=0, ret;
    memset(buf, 0, MAX_MTU_SIZE);

    *(uint16_t *)(buf + buf_size) = htons(TFTP_RRQ);
    buf_size+= 2;
    strcpy((char *)(buf+buf_size), (const char *)filename);
    buf_size += strlen((char *)filename) + 1;
    strcpy((char *)(buf+buf_size), (const char *)TRANS_BINARY);
    buf_size += strlen((char *)TRANS_BINARY) + 1;
    strcpy((char *)(buf+buf_size), (const char *)default_tftp_opt.name);
    buf_size += strlen((char *)default_tftp_opt.name) + 1;
    strcpy((char *)(buf+buf_size), (const char *)default_tftp_opt.value);
    buf_size += strlen((char *)default_tftp_opt.value) + 1;

    ret = packetsend(sn, buf, buf_size, server_ip, TFTP_SERVER_PORT, ip_mode);

    return ret;
}
```

3. 응답이 OACK 패킷이면 Option 처리를 한다.
```c
else if(current_opcode == TFTP_OACK)
                {
                    ret = proc_oack(buf, ret, sn, ip_mode, destip, destport);
}

...

uint16_t send_oack(uint8_t * buf, uint8_t sn, uint8_t ip_mode, uint8_t* destip, uint16_t destport)
{
    uint16_t buf_size, ret;

    memset(buf, 0, MAX_MTU_SIZE);
    buf_size=0;
    *(uint16_t *)(buf + buf_size) = htons(TFTP_OACK);
    buf_size+= 2;
    strcpy((char *)(buf+buf_size), (const char *)"tsize");
    buf_size += strlen("tsize") + 1;

    digittostr(g_tftp_filesize, buf + buf_size);
    buf_size += strlen((char *)(buf + buf_size)) + 1;

    ret = packetsend(sn,buf,buf_size,destip,destport,ip_mode);

    return ret;
}
```

4. 응답이 ERROR 패킷이면 Error 처리를 한다.
```c
else if(current_opcode == TFTP_ERROR)
                {
                    TFTP_ERROR_T *data = (TFTP_ERROR_T *)buf;
                    printf("Error occurred with %s\r\n", data->error_msg);
                    g_tftp_state = STATE_DONE;
                    printf("curr state: STATE_DONE\r\n");
}
```

5. 응답이 DATA 패킷이면 데이터를 수신해서 UART 포트로 출력한다. 이때 수신한 데이터의 길이가 Block size도다 작으면 수신을 종료하고 STATE_DONE으로 상태를 변경한다.
```c
                if(current_opcode == TFTP_DATA)
                {
                    ret = proc_data(buf, sn, ip_mode, destip, destport);

                    if(ret > 0)
                    {
                        if(received_size - 4 < TFTP_BLK_SIZE){
                            g_tftp_state = STATE_DONE;
                            printf("curr state: STATE_DONE\r\n");
                        }
                    }
}

...

uint16_t proc_data(uint8_t * buf, uint8_t sn, uint8_t ip_mode, uint8_t* destip, uint16_t destport)
{
    TFTP_DATA_T *data = (TFTP_DATA_T *)buf;
    uint16_t buf_size = 0;

    uint16_t ret;

    current_block_num = ntohs(data->block_num);

    printf("%s", data->data);

    memset(buf, 0, MAX_MTU_SIZE);

    *(uint16_t *)(buf + buf_size) = htons(TFTP_ACK);
    buf_size+= 2;
    *(uint16_t *)(buf + buf_size) = htons(current_block_num);
    buf_size+= 2;

    return packetsend(sn,buf,buf_size,destip,destport,ip_mode);

}
```


아래 github 주소에 Full Code가 등록되어 있다.

https://github.com/WIZnet-ioLibrary/w6100-evb-gcc-eclipse-tftpc-simple