[<-- До опису бібліотеки](README.md) 

## processS7Packet

```js
function processS7Packet(theData, theItem, thePointer, theCID) {

	var remainingLength;

	if (typeof (theData) === "undefined") {
		remainingLength = 0;
		outputLog("Обробка невизначеного пакета, ймовірно, через помилку тайм-ауту", 0, theCID);
	} else if (isNaN(theItem.byteLength)) {
		// byteLength Nan should probably never reach this method.
		// This temporal fix avoids the library crashing
		outputLog("Processing an undefined packet, perhaps bad input?", 0, theCID);
		return 0;
	} else {
		remainingLength = theData.length - thePointer;  //Скажімо, якщо довжина дорівнює 39, а покажчик(pointer) — 35, ми можемо отримати доступ 35,36,37,38 = 4 bytes.
	}
	var prePointer = thePointer;

	// Створіть новий буфер для якості.
	theItem.qualityBuffer = Buffer.alloc(theItem.byteLength);
	theItem.qualityBuffer.fill(0xFF);  // Заповніть 0xFF (255), що означає NO QUALITY у світі OPC.

	if (remainingLength < 4) {
		theItem.valid = false;
		if (typeof (theData) !== "undefined") {
			theItem.errCode = 'Malformed Packet - Less Than 4 Bytes.  TDL' + theData.length + 'TP' + thePointer + 'RL' + remainingLength;
		} else {
			theItem.errCode = "Timeout error - zero length packet";
		}
		outputLog(theItem.errCode, 0, theCID);
		return 0;   			// Hard to increment the pointer so we call it a malformed packet and we're done.
	}

	var reportedDataLength;

	if (theItem.readTransportCode == 0x04) {
		reportedDataLength = theData.readUInt16BE(thePointer + 2) / 8;  // Для різних транспортних кодів це може бути неправильним.
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
		theItem.errCode = 'Malformed Packet - Item Data Length and Packet Length Disagree.  RDL+2 ' + (reportedDataLength + 2) + ' remainingLength ' + remainingLength;
		outputLog(theItem.errCode, 0 , theCID);
		return 0;   			// Hard to increment the pointer so we call it a malformed packet and we're done.
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
		theItem.errCode = 'Invalid Response Length - Expected ' + expectedLength + ' but got ' + reportedDataLength + ' bytes.';
		outputLog(theItem.errCode, 0 , theCID);
		return reportedDataLength + 2;
	}

	// Looks good so far.
	// Increment our data pointer past the status code, transport code and 2 byte length.
	thePointer += 4;

	theItem.valid = true;
	theItem.byteBuffer = theData.slice(thePointer, thePointer + reportedDataLength);
	theItem.qualityBuffer.fill(0xC0);  // Fill with 0xC0 (192) which means GOOD QUALITY in the OPC world.

	thePointer += theItem.byteLength; //WithFill;

	if (((thePointer - prePointer) % 2)) { // Odd number.  With the S7 protocol we only request an even number of bytes.  So there will be a filler byte.
		thePointer += 1;
	}

	//	outputLog("We have an item value of " + theItem.value + " for " + theItem.addr + " and pointer of " + thePointer);

	return thePointer;
}
```





[<-- До опису бібліотеки](README.md) 





