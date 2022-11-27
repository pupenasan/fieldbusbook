[<-- До опису бібліотеки](README.md) 

## processS7Packet

Обробник отрманих даних `theDat`a для вказаного  `theItem` типу `S7Item`

```js
processS7Packet(theData, theItem, thePointer, theCID)
```

`theData` - пакет даних

`theItem` - елемент

`thePointer` - поточне зміщення в theData 

`theCID` - connectionID

- Перевіряє на коректність отриманих даних
- виставляє якість у `qualityBuffer` та `valid`
- записує отримані дані в `byteBuffer` та кількість отриманих байт в `byteLength`
- збільшує thePointer на `byteLength` з урахуванням вирівнювання

```js
function processS7Packet(theData, theItem, thePointer, theCID) {

	var remainingLength;

	if (typeof (theData) === "undefined") {
		remainingLength = 0;
		outputLog("Обробка невизначеного пакета, ймовірно, через помилку тайм-ауту", 
                  0, theCID);
	} else if (isNaN(theItem.byteLength)) {
		// byteLength Nan, ймовірно, ніколи не досягне цього методу. 
		// Це тимчасове виправлення дозволяє уникнути збою бібліотеки
		outputLog("Обробка невизначеного пакета, можливо, неправильний вхід?", 
                  0, theCID);
		return 0;
	} else {
        // Скажімо, якщо довжина дорівнює 39, а покажчик(pointer) — 35, 
        // ми можемо отримати доступ 35,36,37,38 = 4 bytes.
		remainingLength = theData.length - thePointer;  
	}
	var prePointer = thePointer;

	// Створює новий буфер для якості.
	theItem.qualityBuffer = Buffer.alloc(theItem.byteLength);
    // Заповнює 0xFF (255), що означає NO QUALITY у світі OPC.
	theItem.qualityBuffer.fill(0xFF);  

	if (remainingLength < 4) {
		theItem.valid = false;
		if (typeof (theData) !== "undefined") {
			theItem.errCode = 'Неправильний пакет - менше 4 байтів TDL' 
                + theData.length + 'TP' + thePointer 
                + 'RL' + remainingLength;
		} else {
			theItem.errCode = "Timeout error - zero length packet";
		}
		outputLog(theItem.errCode, 0, theCID);
        // Важко збільшити вказівник, тому ми називаємо його 
        // неправильно сформованим пакетом, і все готово.
		return 0;   			 
	}

	var reportedDataLength;

	if (theItem.readTransportCode == 0x04) {//Deault
        // Для інших транспортних кодів це може бути неправильним.
		reportedDataLength = theData.readUInt16BE(thePointer + 2) / 8;  
	} else {
		reportedDataLength = theData.readUInt16BE(thePointer + 2);
	}
	var responseCode = theData[thePointer];
	var transportCode = theData[thePointer + 1];

	if (remainingLength == (reportedDataLength + 2)) {
		outputLog("Not last part.", 0, theCID);
	}

	if (remainingLength < reportedDataLength + 2) {
		theItem.valid = false;
		theItem.errCode = 'Неправильний пакет - довжина даних елемента' + 
            ' та довжина пакета не збігаються.  RDL+2 ' 
            + (reportedDataLength + 2) + ' remainingLength ' + remainingLength;
		outputLog(theItem.errCode, 0 , theCID);
        // Важко збільшити вказівник, тому ми називаємо його 
        // неправильно сформованим пакетом, і все готово.
		return 0;   			
	}

	if (responseCode !== 0xff) {
		theItem.valid = false;
		theItem.errCode = 'Invalid Response Code - ' + responseCode;
		outputLog(theItem.errCode, 0 , theCID);
		return thePointer + reportedDataLength + 4;
	}

	if (transportCode !== theItem.readTransportCode) {
		theItem.valid = false;
		theItem.errCode = 'Invalid Transport Code - ' + transportCode;
		outputLog(theItem.errCode, 0 , theCID);
		return thePointer + reportedDataLength + 4;
	}

	var expectedLength = theItem.byteLength;

	if (reportedDataLength !== expectedLength) {
		theItem.valid = false;
		theItem.errCode = 'Invalid Response Length - Expected ' 
            + expectedLength + ' but got ' 
            + reportedDataLength + ' bytes.';
		outputLog(theItem.errCode, 0 , theCID);
		return reportedDataLength + 2;
	}

	// Looks good so far.
	// Збільште наш покажчик даних після коду статусу, 
    // транспортного коду та довжини 2 байтів.
	thePointer += 4;

	theItem.valid = true;
	theItem.byteBuffer = theData.slice(thePointer, thePointer + reportedDataLength);
    // Заповнити 0xC0 (192), що означає ДОБРЕ ЯКІСТЬ у світі OPC. 
	theItem.qualityBuffer.fill(0xC0);  

	thePointer += theItem.byteLength; //WithFill;
    //Непарне число. З протоколом S7 ми запитуємо лише парну кількість байтів. 
    // Отже, буде байт-заповнювач.
	if (((thePointer - prePointer) % 2)) { 
		thePointer += 1;
	}

	return thePointer;
}
```

[<-- До опису бібліотеки](README.md) 





