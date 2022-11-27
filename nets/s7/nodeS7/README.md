[S7 TCP/IP](../README.md)

# Бібліотека NodeS7

Даний розділ описує бібліотеку NodeS7, оригінальний репозиторій знаходиться за наступним посиланням:  <https://github.com/plcpeople/nodeS7>

NodeS7 — це бібліотека для Node.js, яка дозволяє зв’язуватися з ПЛК S7-300/400/1200/1500 за допомогою протоколу Siemens S7 Ethernet RFC1006. Це програмне забезпечення жодним чином не пов’язане з Siemens, як і автор. S7-300, S7-400, S7-1200 і S7-1500 є товарними знаками Siemens AG.

## Загальний опис та клієнтський інтерфейс

Даний підрозділ перекладено з [оригінального репозиторію](https://github.com/plcpeople/nodeS7).

### Встановлення

Використовуючи npm:

- `npm install nodes7`

Використовуючи yarn:

- `yarn add nodes7`

### Оптимізація

- Він оптимізований трьома способами - Він сортує велику кількість елементів, які запитуються від ПЛК, і вирішує, які загальні області даних запитувати, а потім групує кілька невеликих запитів разом в один пакет, або групує кілька пакетів до максимальної довжини що підтримує PLC, а потім надсилає кілька пакетів одночасно для максимальної швидкості. Таким чином, запит на 100 різних бітів, усі близькі (але не обов’язково повністю суміжні), буде згруповано в одному запиті до ПЛК без додаткових вказівок від користувача.
- NodeS7 керує повторним підключенням за вас. Отже, якщо з’єднання втрачено через те, що ПЛК вимкнено або відключено, ви можете продовжувати запитувати дані, не потребуючи інших дій. Повертаються значення `Bad`, і зрештою з’єднання буде автоматично відновлено.
- NodeS7 повністю написаний на Javascript, тому не потрібно інсталювати компілятор у Windows, а розгортання на інших платформах (ARM тощо) не повинно бути проблемою.

### Підтримка PLC

- Для доступу до ЦП S7-1200 і S7-1500 потрібен доступ за допомогою «Slot 1», і ви повинні вимкнути оптимізований доступ до блоків (на TIA portal) для блоків, які ви використовуєте. Крім того, на TIA Portal ви повинні "Enable GET/PUT Access" у налаштуваннях контролеру 1200/1500. Це також відкриє контролер для іншого доступу з боку інших програм, тому пам’ятайте про наслідки для безпеки, які це має зробити.
- Це було перевірено на прямих підключеннях до новіших процесорів PROFINET і пристроїв Helmholz NetLINK PRO COMPACT і IBH. (Зауважте, що для цих шлюзів вам часто доводиться вказувати адресу MPI як номер слота). Повідомляється, що він також працює з іншими комбінаціями CPU/CP, хоча підтримуються не всі типи даних S7-200. Маршрутизація S7 не підтримується.
- Підтримуються також ПЛК  `Logo 0BA8` , хоча вам слід налаштувати локальний і віддалений TSAP відповідно до вашого проекту, а ваші адреси потрібно вказати по-різному. `DB1,INT0` має отримати `VM0`. `DB1,INT1118` має отримати `AM1`.

### Підтримка перетворювачів частоти 

Credit to the S7 Wireshark dissector plugin for help understanding why things were not working. (http://sourceforge.net/projects/s7commwireshark/)

- SINAMICS S120 і G120 FW 4.7 і вище також працюють, оскільки ці приводи підтримують пряме підключення ЗА ВИКОРИСТАННЯМ СЛОТА 0 (замість інших прикладів, які використовують 1 або 2) і деяку модифіковану адресацію параметрів. Ця техніка може працювати з цими приводами також з іншим програмним забезпеченням і задокументована на веб-сайті Siemens. По суті, для адресації параметра номер 24, наприклад, вихідна частота, означена в документації як дійсне число, тому `DB24,REAL0` повертатиме вихідну частоту. Якби цей параметр був масивом, `DB24,REAL1` повернув би наступний у послідовності, навіть якщо програміст Siemens мав би спокусу використати `REAL4`, що в даному випадку є неправильним. З цієї причини звичайну оптимізацію S7 потрібно вимкнути. Після оголошення `conn = new nodes7;` (або подібного) додайте `conn.doNotOptimize = true;`, щоб переконатися, що цього не зроблено, і не намагайтеся запитувати ці елементи за допомогою нотації масиву, оскільки це передбачає оптимізацію, запит `REAL0`, потім `REAL1` тощо. Тепер `doNotOptimize` також підтримується як параметр підключення.

Заслуга плагіна S7 Wireshark dissector, який допоміг зрозуміти, чому щось не працює. (http://sourceforge.net/projects/s7commwireshark/)

### Приклади

```js
var nodes7 = require('nodes7'); // Це назва пакета, якщо репозиторій клоновано, вам може знадобитися вимагати «nodeS7» із S у верхньому регістрі
var conn = new nodes7;
var doneReading = false;
var doneWriting = false;

var variables = {
      TEST1: 'MR4',          // Memory real at MD4
      TEST2: 'M32.2',        // Bit at M32.2
      TEST3: 'M20.0',        // Bit at M20.0
      TEST4: 'DB1,REAL0.20', // Array of 20 values in DB1
      TEST5: 'DB1,REAL4',    // Single real value
      TEST6: 'DB1,REAL8',    // Another single real value
      TEST7: 'DB1,INT12.2',  // Two integer value array
      TEST8: 'DB1,LREAL4',   // Single 8-byte real value
      TEST9: 'DB1,X14.0',    // Single bit in a data block
      TEST10: 'DB1,X14.0.8'  // Array of 8 bits in a data block
};
// slot 2 for 300/400, slot 1 for 1200/1500, change debug to true to get more info
conn.initiateConnection({ port: 102, 
                         host: '192.168.56.102', 
                         rack: 0, slot: 2, 
                         debug: true }, connected); 
// conn.initiateConnection({port: 102, host: '192.168.0.2', localTSAP: 0x0100, remoteTSAP: 0x0200, timeout: 8000, doNotOptimize: true}, connected);
// локальний і віддалений TSAP також можна вказати безпосередньо. Опція тайм-ауту вказує час очікування TCP.

function connected(err) {
  if (typeof(err) !== "undefined") {
    // У нас сталася помилка. Можливо, ПЛК недоступний.
    console.log(err);
    process.exit();
  }
  // Це встановлює "переклад", щоб ми могли працювати з назвами об’єктів
  conn.setTranslationCB(function(tag) { return variables[tag]; }); 
  conn.addItems(['TEST1', 'TEST4']);
  conn.addItems('TEST6');
    // conn.removeItems(['TEST2', 'TEST3']); // We could do this.
    // conn.writeItems(['TEST5', 'TEST6'], [ 867.5309, 9 ], valuesWritten); 
    // You can write an array of items as well.
    // conn.writeItems('TEST7', [666, 777], valuesWritten); 
    // You can write a single array item too.
    // This writes a single boolean item (one bit) to true
  conn.writeItems('TEST3', true, valuesWritten); 
  conn.readAllItems(valuesReady);
}

function valuesReady(anythingBad, values) {
  if (anythingBad) { console.log("SOMETHING WENT WRONG READING VALUES!!!!"); }
  console.log(values);
  doneReading = true;
  if (doneWriting) { process.exit(); }
}

function valuesWritten(anythingBad) {
  if (anythingBad) { console.log("SOMETHING WENT WRONG WRITING VALUES!!!!"); }
  console.log("Done writing.");
  doneWriting = true;
  if (doneReading) { process.exit(); }
}
```

### API

- [initiateConnection()](https://github.com/plcpeople/nodeS7#initiate-connection)
- [dropConnection()](https://github.com/plcpeople/nodeS7#drop-connection)
- [setTranslationCB()](https://github.com/plcpeople/nodeS7#set-translation-cb)
- [addItems()](https://github.com/plcpeople/nodeS7#add-items)
- [removeItems()](https://github.com/plcpeople/nodeS7#remove-items)
- [writeItems()](https://github.com/plcpeople/nodeS7#write-items)
- [readAllItems()](https://github.com/plcpeople/nodeS7#read-all-items)

#### nodes7.initiateConnection(options, callback)

Підключається до PLC.

**Arguments**

```js
Options
```

| Property   | type   | default       |
| ---------- | ------ | ------------- |
| rack       | number | 0             |
| slot       | number | 2             |
| port       | number | 102           |
| host       | string | 192.168.8.106 |
| timeout    | number | 5000          |
| localTSAP  | hex    | undefined     |
| remoteTSAP | hex    | undefined     |

```js
callback(err)
```

- `err` - є або об'єктом помилки, або undefined у разі успішного підключення. 

#### nodes7.dropConnection(callback)

Відключається від ПЛК. Це просто розриває з'єднання TCP.

**Arguments**

```
callback()
```

Зворотний виклик викликається після завершення закриття.

#### nodes7.setTranslationCB(translator)

Встановлює зворотний виклик для перекладу імені та адреси.

Це необов’язково – ви можете використовувати `addItem` тощо з абсолютними адресами.

Якщо ви його використовуєте, `translator` має бути функцією, яка приймає рядок як аргумент і повертає рядок у такому форматі: `<data block number.><memory area><data type><byte offset><.array length>`

**Приклади**:

- MR30 - MD30 as REAL
- DB10,LR32 - LREAL at byte offset 32 in DB10, for 1200/1500 only
- DB10,INT6 - DB10.DBW6 as INT
- DB10,I6 -same as above
- DB10,INT6.2 - DB10.DBW6 and DB10.DBW8 in an array with length 2
- DB10,X14.0 - DB10.DBX14.0 as BOOL
- DB10,X14.0.8 - DB10.DBB14 as an array of 8 BOOL
- PIW30 - PIW30 as INT
- DB10,S20.30 - String at offset 20 with length of 30 (actual array  length 32 due to format of String type, length byte will be  read/written)
- DB10,S20.30.3 - Array of 3 strings at offset 20, each with length of 30 (actual array length 32 due to format of String type, length byte  will be read/written)
- DB10,C22.30 - Character array at offset 22 with length of 30 (best to not use this with strings as length byte is ignored)
- DB10,DT0 - Date and time
- DB10,DTZ0 - Date and time in UTC
- DB10,DTL0 - DTL in newer PLCs
- DB10,DTLZ0 - DTL in newer PLCs in UTC

Тип `DT` — це добре відомий тип `DATE_AND_TIME` ПЛК S7-300/400, поле шириною 8 байт із частинами, закодованими у BCD

Тип `DTZ` такий самий, як і `DT`, але він очікує, що мітка часу в ПЛК у форматі UTC (зазвичай це НЕ так)

Тип `DTL` – це той, який можна побачити на нових ПЛК S7-1200/1500, має довжину 12 байт і кодує мітку часу інакше, ніж старіший `DATE_AND_TIME`

Тип `DTLZ` також такий самий, як і `DTL`, але очікується позначка часу в UTC у ПЛК

У наведеному вище прикладі оголошено об’єкт, а  `translator` посилається на цей об’єкт. Це може просто посилатися на файл або базу даних. У будь-якому випадку це дозволяє писати чистіший код Javascript, який посилається на ім’я замість абсолютної адреси.

#### nodes7.addItems(items)

Додає `items` до внутрішнього списку опитувань читання.

**Аргументи**

`items` може бути string або масивом string . Якщо `items` містить значення `_COMMERR`, воно поверне поточний статус зв’язку.

#### nodes7.removeItems(items)

Видаляє `items` з внутрішнього списку опитувань читання.

**Аргументи**

`items` може бути string або масивом string .  Якщо `items` не визначено, усі елементи буде видалено.

#### nodes7.writeItems(items, values, callback)

Записує `items` до ПЛК, використовуючи відповідні `values`, і після завершення викликає `callback`. Ви повинні стежити за поверненим значенням - якщо воно відмінне від нуля, запис не буде оброблено, оскільки він вже триває, і зворотний виклик не буде викликано.

**Аргументи**

`items` може бути string або масивом string. Якщо `items` є одним string , `values` має бути одним елементом. Якщо `items` є масивом string , `values` також має бути масивом значень.

```
callback(err)
```

- err - логічне значення, яке вказує, чи БУДЬ-ЯКИЙ із елементів має "bad quality".

#### nodes7.readAllItems(callback)

Читає внутрішній список опитувань і після завершення викликає `callback`.

**Аргументи**

```
callback(err, values)
```

- err - логічне значення, що вказує, чи БУДЬ-ЯКИЙ із елементів має "погану якість".
- values - об'єкт, що містить значення, що зчитуються як ключі, і їх значення (з ПЛК) як значення.

## Методи

Даний розділ та інші файли папки репозиторію розібрані та прокоментовані вже мною.

- [bufferizeS7Item](bufferizeS7Item.md)
- [checkRFCData](checkRFCData.md)
- [doneSending](doneSending.md)
- [fromBCD](fromBCD.md)
- [getWriteBuffer](getWriteBuffer.md)
- [isQualityOK](isQualityOK.md)
- [itemListSorter](itemListSorter.md) - Сортує елементи масиву з S7Item по пріоритетності
- [outputLog(txt, debugLevel, id)](outputLog.md) - Виводить в консоль повідомлення `txt` з вказаним `id` та рівнем критичності `debugLevel`
- [processS7Packet](processS7Packet.md)
- [processS7ReadItem](processS7ReadItem.md)
- [processS7WriteItem](processS7WriteItem.md)
- [prototype.addItems (arg)](prtp.addItems.md) - 
- [prototype.addItemsNow](prtp.addItemsNow.md)
- [prototype.clearReadPacketTimeouts](prtp.clearReadPacketTimeouts.md)
- [prototype.clearWritePacketTimeouts](prtp.clearWritePacketTimeouts.md)
- [prototype.connectError](prtp.connectError.md)
- [prototype.connectionCleanup](prtp.connectionCleanup.md)
- [prototype.connectionReset](prtp.connectionReset.md) - Обнуляє стан зєднання (`isoConnectionState=0`) і якщо немає ніяких запитів запускається таймер на 3.5 с після чого запускається  функція `resetNow`
- [prototype.connectNow (cParam)](prtp.connectNow.md) - функція викликається в initiateConnection, реалізує з'єднання 
- [prototype.dropConnection](prtp.dropConnection.md)
- [prototype.findItem](prtp.findItem.md)
- [prototype.findReadIndexOfSeqNum](prtp.findReadIndexOfSeqNum.md)
- [prototype.findWriteIndexOfSeqNum](prtp.findWriteIndexOfSeqNum.md)
- [prototype.getNextSeqNum](prtp.getNextSeqNum.md)
- [prototype.initiateConnection (cParam, callback)](prtp.initiateConnection.md) - підключеється до ПЛК з адресою та параметрами 
- [prototype.isOptimizableArea](prtp.isOptimizableArea.md)
- [prototype.isReading()](prtp.isReading.md) - Показує чи бібліотека в процесі читання даних
- [prototype.isWaiting()](prtp.isWaiting.md) - показує чи бібліотека зайнята читанням чи записом
- [prototype.isWriting()](prtp.isWriting.md) - Показує чи бібліотека в процесі запису даних
- [prototype.onClientClose](prtp.onClientClose.md)
- [prototype.onClientDisconnect](prtp.onClientDisconnect.md)
- [prototype.onISOConnectReply](prtp.onISOConnectReply.md)
- [prototype.onPDUReply](prtp.onPDUReply.md)
- [prototype.onResponse](prtp.onResponse.md)
- [prototype.onTCPConnect ()](prtp.onTCPConnect.md) - Викликається при вдалому підключенні по TCP, для реалізації підключення по ISO/TCP
- [prototype.packetTimeout (packetType, packetSeqNum)](prtp.packetTimeout.md) - функція зворотного виклику при спрацюванню таймауту пакету. Виводить діагностичні повідомлення в журнал і робить певні дії в залежності від типу пакету. 
- [prototype.prepareReadPacket](prtp.prepareReadPacket.md)
- [prototype.prepareWritePacket](prtp.prepareWritePacket.md)
- [prototype.readAllItems](prtp.readAllItems.md)
- [prototype.readResponse](prtp.readResponse.md)
- [prototype.readWriteError](prtp.readWriteError.md)
- [prototype.removeItems](prtp.removeItems.md)
- [prototype.removeItemsNow](prtp.removeItemsNow.md)
- [prototype.resetNow](prtp.resetNow.md) - Обнуляє стан зєднання (`isoConnectionState = 0`), та закриває зєднання
- [prototype.sendReadPacket](prtp.sendReadPacket.md)
- [prototype.sendWritePacket](prtp.sendWritePacket.md)
- [prototype.setTranslationCB (cb)](prtp.setTranslationCB.md) - Встановлює функцію зворотного виклику `cb` при перетворенні.
- [prototype.writeItems (arg, value, cb)](prtp.writeItems.md) - Формує масив `instantWriteBlockList` з елементів типу `S7Item` що записуються та їх значень плся чого викликає `prepareWritePacket` для ...  та `sendWritePacket` для.
- [prototype.writeResponse](prtp.writeResponse.md)
- [readDT](readDT.md)
- [readDTL](readDTL.md)
- [S7AddrToBuffer](S7AddrToBuffer.md)
- [S7Item](S7Item.md)
- [S7Packet](S7Packet.md)
- [stringToS7Addr (addr, useraddr, cParam)](stringToS7Addr.md) - Повертає обєкт `S7Item` за текстовим представленням
- [toBCD](toBCD.md)
- [writeDT](writeDT.md)
- [writeDTL](writeDTL.md)
- [writePostProcess](writePostProcess.md)

## Властивості

- addRemoveArray - Масив дій з елементами

  ```json
  {arg: arg, 
  action: 'add' //дії add, remove,
  }
  ```

- connectCallback

- connectCBIssued

- connectionID

- connectionParams

- connectReq - буфер запиту на підключення по ISO/TCP

- connectTimeout - таймер, який виконує функцію при таймауту пакету

- doNotOptimize

- dropConnectionCallback

- dropConnectionTimer

- effectiveDebugLevel = opts.debug ? 99 : 0

- globalReadBlockList - глобальний список зчитуваних елементів що містять елементи S7Item

- globalTimeout - 

- globalWriteBlockList - глобальний список записуваних елементів що містять елементи S7Item

- instantWriteBlockList - Список `S7Item` для запису.

- isoclient - клієнтський сокет 

- isoConnectionState - 

  - 0 = no connection

  - 1 = trying to connect

  - 2 = TCP connected, wait for ISO connection confirmation

  - 3 = ISO-ON-TCP connected, Wait for PDU response

  - 4 = Received PDU response, good to go

- localTSAP

- masterSequenceNumber

- maxGap

- maxParallel

- maxPDU - параметри для переговорів при підключенні

- negotiatePDU - буфер з даних запиту, що відправляються після відповіді-підтвердження на з'єднання, переговори по S7 

- opts - опції

- parallelJobsNow

- PDUTimeout - 

- polledReadBlockList - Масив елментів для читання

- rack

- readDoneCallback

- readPacketArray - масив пакетів на читання

  - `seqNum` - номер пакету

  - `sent` - TRUE - пакет відправлено

  - `rcvd` - TRUE - відповідь на пакет отримано
  
  - `itemList` - масив `S7Item`
  
  
    - `timeoutError`
  


- readPacketValid
- readReq - Bufferr(1500) для відправлення запиту 
- readReqHeader
- remoteTSAP
- requestMaxParallel - параметри для переговорів при підключенні
- requestMaxPDU - параметри для переговорів при підключенні
- rereadTimer
- resetPending
- resetTimeout - Timed reset has happened (connectionReset)
- silentMode = opts.silent || false
- slot
- translationCB - Функція зворотного виклику перетворення символьних імен в адреси. Див також `setTranslationCB`
- writeDoneCallback - Функція зворотного виклику для читання або запису
- writeInQueue
- writePacketArray  - масив пакетів на запис


  - `seqNum` - номер пакету
  - `sent` - TRUE - пакет відправлено

  - `rcvd` - TRUE - відповідь на пакет отримано

  - `itemList` - масив `S7Item`


    - `timeoutError`

- writeReq
- writeReqHeader

## Класи 

### S7Item

**Властивості**:

`addr` - адреса в форматі S7, наприклад `DB1,X14.0`

`useraddr` - користувацька навза елементу

`addrtype` - тип елементу S7:

- DB
- P - периферійних опеірацій вводу-виводу
- I - 
- Q - 
- M - 
- T - 
- C - 

`datatype` - тип даних S7:

- DT - Date
- DTZ - Date
- DTL - Date
- DTLZ - Date
- REAL
- LREAL
- DWORD
- DINT
- INT
- LINT
- WORD
- B
- BYTE
- TIMER
- COUNTER
- X - bool
- C
- CHAR
- S
- STRING

`dbNumber` - номер DB, наприклад 1 (`TEST10: 'DB1,X14.0'` )

`bitOffset` - адреса біту,  14

`offset` - адреса байту,  14

`arrayLength` - довжина масиву 

`dtypelen` - довина в байтах емелементу масиву

`areaS7Code` - область даних в S7

-  `0x84` - DB, DI
- `0x81` - I
- `0x82` - Q
- `0x83` - M
- `0x80` - P
- `0x1c` - C
- `0x1d` - T

`byteLength` - загальний обєм в байтах

`byteLengthWithFill ` - вирівняний обєм  в байтах

`readTransportCode` - 

-  `0x04` - Deault
- `0x09` - для areaS7Code: 
  - C
  - T

`writeTransportCode` - повторює `readTransportCode`, з винятком: 

- `0x03` - для datatype === 'X'

`byteBuffer` - сирі прочитані дані 

`writeBuffer`

`qualityBuffer` - буфер з заповненими значеннями якосіт для даних:

- `0xFF` - BAD
- `0xC0` - Good

`writeQualityBuffer` 

`value` - значення

`writeValue`

`valid` - прийшли валідні дані 

`errCode`  - код помилки в текстовому вигляді

`part`

`maxPart` 

`isOptimized` 

`resultReference` 

`itemReference` 



**Методи**:

`clone` - клонування об'єкту 

`badValue` - повертає значення по дефолту в залежності від типу обєкту datatype

## Принципи роботи

### Ініціювання підключення

З точки зору користувача:

- спочатку створюється об'єкт типу `nodes7`, наприклад `conn`
- викликається метод `initiateConnection` з передачею параметрів підключення та функцією зворотного виклику наприклад `connected`
- у функції зворотного виклику прописується усі операції, які робляться після підключення

```js
var nodes7 = require('nodes7'); // Це назва пакета, якщо репозиторій клоновано, вам може знадобитися вимагати «nodeS7» із S у верхньому регістрі
var conn = new nodes7;
// slot 2 for 300/400, slot 1 for 1200/1500, change debug to true to get more info
conn.initiateConnection({ port: 102, 
                         host: '192.168.56.102', 
                         rack: 0, slot: 2, 
                         debug: true }, connected); 

function connected(err) {
  if (typeof(err) !== "undefined") {
    // У нас сталася помилка. Можливо, ПЛК недоступний.
    console.log(err);
    process.exit();
  }
  // Далі використання conn для читання та запису
  conn.setTranslationCB(function(tag) { return variables[tag]; }); 
  conn.addItems(['TEST1', 'TEST4']);
  conn.writeItems('TEST3', true, valuesWritten); 
  conn.readAllItems(valuesReady);
  //...  
}
```

Функція [initiateConnection](prtp.initiateConnection.md) :

- заповняє властивості підключення за замовченням, якщо вони не вказані
- викликає функцію [connectNow](prtp.connectNow.md), яка реалізує саме з'єднання:
  - викликає [connectionCleanup](prtp.connectionCleanup.md) для очистки попереднє з'єднання сокету `isoclient`
  - створює новий сокет `isoclient = net.connect(cParam);`
  - як функцію зворотного виклику при створенні TCP підключення вказує [onTCPConnect](prtp.onTCPConnect.md), який реалізовує підключення по ISO/TCP, а саме:
    - `isoConnectionState = 2` -  TCP підключено, чекаємо підключення ISO
    - ставиться обробник помилки таймауту [packetTimeout](prtp.packetTimeout.md)
    - `connBuf = self.connectReq.slice();` - де `connectReq` - масив з байтів запиту на підключення 
    - записує в `connBuf` `localTSAP`, `remoteTSAP`
    - Запускає метод `write` сокету `isoclient`
    - прослушка на подію 'data' - [onISOConnectReply](prtp.onISOConnectReply.md), яка викликається при отриманні даних, вона:
      - `isoConnectionState = 3` - ISO-ON-TCP connected, Wait for PDU response
      - якщо неочікувані довжини отрманих даних, то [connectionReset](prtp.connectionReset.md) - обнуляє стан з'єднання
      - Записує в `negotiatePDU`  (буфер запиту переговорів по S7) 
      - Відправляє пакет `isoclient.write`
      - прослушка на подію 'data' - [onPDUReply](prtp.onPDUReply.md), в якій приходить відповідь з переговорів:
        - отримує оброблені дані з  [checkRFCData](checkRFCData.md) `data=checkRFCData(theData);`
        - якщо `data==="fastACK"` (старі ПЛК), повторно очікуємо прослушки, щоб доотримати порцію даних, що залишилася
        - якщо пакет норм, то за результатами відповіді береться `maxParallel` та `maxPDU`, `self.isoConnectionState = 4` -  4 = Received PDU response, good to go; включається прослушка на подію 'data': [onResponse](prtp.onResponse.md) (див нижче опис)
      - прослушка на подію 'error' - [readWriteError](prtp.readWriteError.md)
    - прослушка на подію 'end' - [onClientDisconnect](prtp.onClientDisconnect.md)
    - прослушка на подію 'close' - [onClientClose](prtp.onClientClose.md)
  - як функцію зворотного виклику при помилці TCP  підключення вказує [connectError](prtp.connectError.md), яка викличе функцію зворотного виклику, наприклад `connected` з передачею відповідної помилки 



### Читання 

ToDo

### Запис 

Запис відбувається викликом [writeItems](prtp.writeItems.md). 

```js
writeItems (arg, value, cb)
```

`arg` - назва змінної, або масив назв для запису 

`value` - значення для запису, або масив значень для запису

`cb` - функція зворотного виклику для виклику після запису

ToDo

```js
var nodes7 = require('nodes7'); 
var conn = new nodes7;
var doneWriting = false;

var variables = {
      TEST3: 'M20.0',        // Bit at M20.0
};
conn.initiateConnection({ port: 102, 
                         host: '192.168.56.102', 
                         rack: 0, slot: 2, 
                         debug: true }, connected); 

function connected(err) {
  if (typeof(err) !== "undefined") {
    // У нас сталася помилка. Можливо, ПЛК недоступний.
    console.log(err);
    process.exit();
  }
  // Це встановлює "переклад", щоб ми могли працювати з назвами об’єктів
  conn.setTranslationCB(function(tag) { return variables[tag]; }); 
  conn.writeItems('TEST3', true, valuesWritten); 
}

function valuesWritten(anythingBad) {
  if (anythingBad) { console.log("SOMETHING WENT WRONG WRITING VALUES!!!!"); }
  console.log("Done writing.");
  doneWriting = true;
  if (doneReading) { process.exit(); }
}
```



### Прослушка на отримання даних

Після підключення ставиться прослушка на подію 'data':  [onResponse](prtp.onResponse.md), яка обробляє отримані вхідні дані:

- якщо довжина маленька - скидує з'єднання [connectionReset](prtp.connectionReset.md) 
- перевіряє `data=checkRFCData(theData);`, (старі ПЛК), повторно очікуємо прослушки, щоб доотримати порцію даних, що залишилася
- перевіряє, якщо довжина не співпадає - скидує з'єднання [connectionReset](prtp.connectionReset.md) 
- перевіряє, якщо довжина велика - фрагментує
- шукає номер пакету в [findReadIndexOfSeqNum](prtp.findReadIndexOfSeqNum.md), якщо знаходить то  `isReadResponse = true;`, виклик [readResponse (data, foundSeqNum)](prtp.readResponse.md):
  - якщо пакет не відправлявся `!sent` або обробка вже відбувалася`rcvd==TRUE` тоді повернути `null`
  - переобор по `itemList` для кожного з яких вилкикати [processS7Packet(data, itemList[itemCount], dataPointer, connectionID)](processS7Packet.md), який:
    - перевіряє на валідність шматок
    - виставляє якість у `qualityBuffer` та `valid`
    - записує отримані дані в `byteBuffer` та кількість отриманих байт в `byteLength`
    - збільшує thePointer на `byteLength` з урахуванням вирівнювання 
  - якщо кожен елемент масиву з пакету оброблено `readPacketArray.every(doneSending)`([doneSending](doneSending.md)) тоді 
    - проходить по елементам `globalReadBlockList` і для кожного елементу
      - обробляє усі requestReference і ...
      - обробляє усі itemReference для яких
        - обробляє [processS7ReadItem](processS7ReadItem.md)
      - для всіх у кого quality=good 
  - інакше `sendReadPacket()`
- шукає номер пакету в [findWriteIndexOfSeqNum](prtp.findWriteIndexOfSeqNum.md), якщо знаходить то  `isWriteResponse = true;`виклик [writeResponse (data, foundSeqNum)](prtp.writeResponse.md), який:
  - ToDo


## Загальний код

```js
var net = require("net");
var util = require("util");
var effectiveDebugLevel = 0; // intentionally global, shared between connections
var silentMode = false;

module.exports = NodeS7;

function NodeS7(opts) {
	opts = opts || {};
	silentMode = opts.silent || false;
	effectiveDebugLevel = opts.debug ? 99 : 0

	var self = this;

	self.connectReq = Buffer.from([0x03, 0x00, 0x00, 0x16, 0x11, 0xe0, 0x00, 0x00, 0x00, 0x02, 0x00, 0xc0, 0x01, 0x0a, 0xc1, 0x02, 0x01, 0x00, 0xc2, 0x02, 0x01, 0x02]);
	self.negotiatePDU = Buffer.from([0x03, 0x00, 0x00, 0x19, 0x02, 0xf0, 0x80, 0x32, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x08, 0x00, 0x00, 0xf0, 0x00, 0x00, 0x08, 0x00, 0x08, 0x03, 0xc0]);
	self.readReqHeader = Buffer.from([0x03, 0x00, 0x00, 0x1f, 0x02, 0xf0, 0x80, 0x32, 0x01, 0x00, 0x00, 0x08, 0x00, 0x00, 0x0e, 0x00, 0x00, 0x04, 0x01]);
	self.readReq = Buffer.alloc(1500);
	self.writeReqHeader = Buffer.from([0x03, 0x00, 0x00, 0x1f, 0x02, 0xf0, 0x80, 0x32, 0x01, 0x00, 0x00, 0x08, 0x00, 0x00, 0x0e, 0x00, 0x00, 0x05, 0x01]);
	self.writeReq = Buffer.alloc(1500);

	self.resetPending = false;
	self.resetTimeout = undefined;//Timed reset has happened (connectionReset)
	self.isoclient = undefined; // клієнтський сокет 
	self.isoConnectionState = 0;     
	self.requestMaxPDU = 960;
	self.maxPDU = 960;
	self.requestMaxParallel = 8;
	self.maxParallel = 8;
	self.parallelJobsNow = 0;
	self.maxGap = 5;
	self.doNotOptimize = false;
	self.connectCallback = undefined;
	self.readDoneCallback = undefined;
	self.writeDoneCallback = undefined;
	self.connectTimeout = undefined;
    //таймер, який виконує функцію при таймауту пакету
	self.PDUTimeout = undefined;
	self.globalTimeout = 1500; // In many use cases we will want to increase this
    // In 0.3.17 this was made variable from cParam.timeout to ensure packets 
    // don't timeout at 1500ms if the user has specified a timeout externally.
	self.rack = 0;
	self.slot = 2;
	self.localTSAP = null;
	self.remoteTSAP = null;

	self.readPacketArray = [];
	self.writePacketArray = [];
	self.polledReadBlockList = [];
	self.instantWriteBlockList = [];
	self.globalReadBlockList = [];
	self.globalWriteBlockList = [];
	self.masterSequenceNumber = 1;
	self. = doNothing;
	self.connectionParams = undefined;
	self.connectionID = 'UNDEF';
	self.addRemoveArray = [];
	self.readPacketValid = false;
	self.writeInQueue = false;
	self.connectCBIssued = false;
	self.dropConnectionCallback = null;
	self.dropConnectionTimer = null;
	self.reconnectTimer = undefined;
    //таймер повторного підключення по ISO/TCP
	self.rereadTimer = undefined;
}

function doNothing(arg) {
	return arg;
}
```

## Ліцензія

```js
// NodeS7 - A library for communication to Siemens PLCs from node.js.

// The MIT License (MIT)

// Copyright (c) 2013 Dana Moffit

// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:

// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.

// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
// THE SOFTWARE.

// EXTRA WARNING - This is BETA software and as such, be careful, especially when
// writing values to programmable controllers.
//
// Some actions or errors involving programmable controllers can cause injury or death,
// and YOU are indicating that you understand the risks, including the
// possibility that the wrong address will be overwritten with the wrong value,
// when using this library.  Test thoroughly in a laboratory environment.
```

