# Протокол S7 TCP/IP

https://support.industry.siemens.com/cs/document/26483647/what-properties-advantages-and-special-features-does-the-s7-protocol-offer-?dti=0&lc=en-WW

All SIMATIC S7 CPUs and C7 CPUs have integrated S7 communication services with which the user program can read and write data.

Усі процесори SIMATIC S7 і процесори C7 мають інтегровані служби зв’язку S7, за допомогою яких програма користувача може читати та записувати дані.

The following functions are available to you for the S7 CPUs and C7 CPUs  regardless of the bus system used, so that you can use S7 communication  via Industrial Ethernet, PROFIBUS or MPI:

- System function blocks (SFBs): in STEP 7 V5.x for S7-400 CPUs 
- Function blocks (FBs): in STEP 7 V5.x for S7-300 CPUs and C7-CPUs
- Instructions: in TIA Portal for S7-300 CPUs, S7-400 CPUs, S7-1200 CPUs and S7-1500 CPUs

Наступні функції доступні для процесорів S7 і C7 незалежно від використовуваної системи шини, щоб ви могли використовувати зв’язок S7 через Industrial Ethernet, PROFIBUS або MPI:

- Системні функціональні блоки (SFB): у STEP 7 V5.x для ЦП S7-400
- Функціональні блоки (FB): у STEP 7 V5.x для процесорів S7-300 і C7-CPU
- Інструкції: на TIA Portal для ЦП S7-300, ЦП S7-400, ЦП S7-1200 і ЦП S7-1500

Position of the S7 protocol in the ISO-OSI reference model.

Позиція протоколу S7 в еталонній моделі ISO-OSI.![img](media/NET_S7_Protokoll_01.png)
Fig. 1 

### Services of the S7 protocol

Overview of the S7 protocol services.
Огляд послуг протоколу S7. 

Table 1

| Service      | Description                                                  |
| :----------- | :----------------------------------------------------------- |
| PUT / GET    | This service is a unidirectional read/write service for transferring small volumes of data to and from a station.<br />Ця послуга є однонаправленою службою читання/запису для передачі невеликих обсягів даних на станцію та зі станції. |
| BSEND / BRCV | This service is a bidirectional and block-oriented service for transferring large volumes of data between two stations.<br />Ця послуга є двонаправленою та блоково-орієнтованою службою для передачі великих обсягів даних між двома станціями |
| USEND / URCV | This service is a bidirectional and uncoordinated service for transferring small volumes of data between two stations.<br />Ця послуга є двонаправленою та некоординованою послугою для передачі невеликих обсягів даних між двома станціями. |

### User data size

The S7 protocol permits transfer of data from 1 byte to 64 Kbytes. The  maximum data size depends on the service used and the S7 CPU used.
Протокол S7 дозволяє передавати дані від 1 байта до 64 Кбайт. Максимальний розмір даних залежить від використовуваної служби та процесора S7.

Table 2

| Service      | S7-300 CPU                | S7-400 CPU  | S7-1200 CPU | S7-1500 CPU                                                  |
| :----------- | :------------------------ | :---------- | :---------- | :----------------------------------------------------------- |
| PUT / GET    | 160 bytes                 | 400 bytes   | 160 bytes   | 880 bytes                                                    |
| BSEND / BRCV | 32768 bytes / 65534 bytes | 65534 bytes | -           | 65534 bytes with standard access65535 bytes with optimized access |
| USEND / URCV | 160 bytes                 | 440 bytes   | -           | 920 bytes                                                    |

### Properties of the S7 protocol

У наступній таблиці показано властивості протоколу S7.

Table 3

| Properties                    | PUT / GET                                 | BSEND / BRCV                            | USEND / URCV                        |
| :---------------------------- | :---------------------------------------- | :-------------------------------------- | :---------------------------------- |
| Memory areas                  | M, D, E, A, T, Z                          | M, D, E, A, T, Z                        | M, D, E, A, T, Z                    |
| Data consistency              | 8 to 32 bytes32 bytes to total length1)2) | Total length per job2)                  | Total length per job2)              |
| Communication principle       | Client / Server                           | Client / Client                         | Client / Client                     |
| Maximum number of connections | See CPU specification                     | See CPU specification                   | See CPU specification               |
| Functions                     | FB15 / SFB15 "PUT"FB14 / SFB14 "GET"      | FB12 / SFB12 "BSEND"FB13 / SFB13 "BRCV" | FB8 / SFB8 "USEND"FB9 / SFB9 "URCV" |

1) Depending on the CPU used. 

2) In the user program you must make sure that the data block is not modified during data transfer.

### Advantages of the S7 protocol

- Independent of the bus medium (PROFIBUS, Industrial Ethernet, MPI). 
- Can be used on all S7 data areas. 
- Transfer of up to 64 Kbytes in one job. 
- The S7 protocol ensures automatic acknowledgment of the data records.
- Low processor and bus load during transfer of large volumes of data.



- Незалежність від середовища передачі (PROFIBUS, Industrial Ethernet, MPI).
- Можна використовувати для всіх областей даних S7.
- Передача до 64 Кбайт за одне завдання.
- Протокол S7 забезпечує автоматичне підтвердження записів даних.
- Низьке навантаження на процесор і шину при передачі великих обсягів даних.

The S7 protocol is supported by all available S7 CPUs and communication  processors. Furthermore, PC systems with appropriate hardware and  software support communication via the S7 protocol. 

Протокол S7 підтримується всіма доступними ЦП і комунікаційними процесорами S7. Крім того, комп'ютерні системи з відповідним апаратним і програмним забезпеченням підтримують зв'язок через протокол S7.

##  nodeS7 в node.js

https://github.com/plcpeople/nodeS7



## S7 Communication (S7comm) WireShark

[S7 Communication (S7comm) WireShark WiKi](https://wiki.wireshark.org/S7comm)

S7comm (S7 Communication) — це власний протокол Siemens, який працює між програмованими логічними контролерами (PLC) сімейства Siemens S7-300/400. Він використовується для програмування ПЛК, обміну даними між ПЛК, доступу до даних ПЛК із систем SCADA (диспетчерського керування та збору даних) і діагностичних цілей.

Дані S7comm надходять як корисне навантаження пакетів даних COTP. Перший байт завжди `0x32` як ідентифікатор протоколу. Спеціальні комунікаційні процесори для серії S7-400 (CP 443) можуть використовувати цей протокол без рівнів TCP/IP.

|      | OSI layer          | Protocol              |
| ---- | ------------------ | --------------------- |
| 7    | Application Layer  | S7 communication      |
| 6    | Presentation Layer | S7 communication      |
| 5    | Session Layer      | S7 communication      |
| 4    | Transport Layer    | ISO-on-TCP (RFC 1006) |
| 3    | Network Layer      | IP                    |
| 2    | Data Link Layer    | Ethernet              |
| 1    | Physical Layer     | Ethernet              |

Для встановлення з’єднання з ПЛК S7 потрібно виконати 3 кроки:

1. Підключіться до ПЛК через TCP-порт 102
2. Підключення на рівні ISO (запит на підключення COTP)
3. Підключіться до рівня S7comm (`s7comm.param.func = 0xf0`, налаштування зв’язку)

Крок 1) використовує IP-адресу PLC/CP.

Крок 2) використовує як пункт призначення TSAP довжиною два байти. Перший байт TSAP призначення кодує тип зв’язку (`1=PG`, `2=OP`). Другий байт цільового TSAP кодує стійку та номер слота: це позиція центрального процесора PLC. Номер слота кодується в бітах `0-4`, номер стійки - в бітах `5-7`.

Крок 3) призначений для узгодження конкретних деталей S7comm (наприклад, розмір PDU).

### Історія

Цей протокол використовується Siemens з моменту запуску серії продуктів Simatic S7 у 1994 році. Протокол також використовується поверх інших фізичних/мережевих рівнів, таких як RS-485 з MPI (Multi-Point-Interface) або Profibus.

### Залежності протоколу

S7-комунікації складається (принаймні) з таких протоколів:

- [COTP](https://wiki.wireshark.org/COTP): ISO 8073 COTP Connection-Oriented Transport Protocol (spec. available as [RFC905](http://www.ietf.org/rfc/rfc0905.txt))
- [TPKT](https://wiki.wireshark.org/TPKT): [RFC1006](http://www.ietf.org/rfc/rfc1006.txt) "ISO transport services on top of the TCP: Version 3", updated by RFC2126
- [TCP](https://wiki.wireshark.org/TCP): Typically, TPKT uses [TCP](https://wiki.wireshark.org/TCP) as its transport protocol. The well known TCP port for TPKT traffic is 102.

### Приклад трафіку

![S7comm_traffic_example.png](https://wiki.wireshark.org/uploads/__moin_import__/attachments/S7comm/S7comm_traffic_example.png)

### Wireshark

Диссектор S7comm частково справний.

### Налаштування параметрів

(XXX додає посилання на налаштування параметрів, які впливають на те, як розбирається PROTO).

### Example capture file

- [SampleCaptures/s7comm_downloading_block_db1.pcap](https://wiki.wireshark.org/uploads/__moin_import__/attachments/SampleCaptures/s7comm_downloading_block_db1.pcap) s7comm: connecting and downloading program block DB1 into PLC
- [SampleCaptures/s7comm_program_blocklist_onlineview.pcap](https://wiki.wireshark.org/uploads/__moin_import__/attachments/SampleCaptures/s7comm_program_blocklist_onlineview.pcap) s7comm: connecting and getting a list of all available block in the PLC
- [SampleCaptures/s7comm_reading_plc_status.pcap](https://wiki.wireshark.org/uploads/__moin_import__/attachments/SampleCaptures/s7comm_reading_plc_status.pcap) s7comm: connecting and viewing the PLC status
- [SampleCaptures/s7comm_reading_setting_plc_time.pcap](https://wiki.wireshark.org/uploads/__moin_import__/attachments/SampleCaptures/s7comm_reading_setting_plc_time.pcap) s7comm: connecting, reading and setting the time of the PLC
- [SampleCaptures/s7comm_varservice_libnodavedemo.pcap](https://wiki.wireshark.org/uploads/__moin_import__/attachments/SampleCaptures/s7comm_varservice_libnodavedemo.pcap) s7comm: running libnodave demo with S7-300 PLC, using variable-services with several areas
- [SampleCaptures/s7comm_varservice_libnodavedemo_bench.pcap](https://wiki.wireshark.org/uploads/__moin_import__/attachments/SampleCaptures/s7comm_varservice_libnodavedemo_bench.pcap) s7comm: running libnodave demo benchmark with S7-300 PLC using variable-services to check the communication capabilities

### Display Filter

A complete list of PROTO display filter fields can be found in the [display filter reference](https://www.wireshark.org/docs/dfref/s/s7comm.html)

Show only the S7comm based traffic:

```
 s7comm 
```

### Capture Filter

You cannot directly filter S7comm protocols while capturing.

S7comm uses port 102, so it is possible to capture S7comm data by using the capture filter

```
tcp port 102 
```

### External links

- [RFC1006](http://www.ietf.org/rfc/rfc1006.txt) *ISO Transport Service on top of the TCP Version: 3*, based on ISO 8073
- [RFC905](http://www.ietf.org/rfc/rfc0905.txt) *ISO Transport Protocol Specification ISO DP 8073*
- [Siemens - Information about the properties of the S7 protocol](https://support.industry.siemens.com/cs/ww/en/view/26483647) *What properties, advantages and special features does the S7 protocol offer* - Siemens Industry Online Support

