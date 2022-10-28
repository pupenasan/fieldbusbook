# Протокол S7 TCP/IP

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