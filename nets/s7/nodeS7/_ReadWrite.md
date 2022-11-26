# Всокорівневі функції Read/Write 

## readWriteError

```js
NodeS7.prototype.readWriteError = function(e) {
	var self = this;
	outputLog('We Caught a read/write error ' + e.code + ' - will DISCONNECT and attempt to reconnect.');
	self.isoConnectionState = 0;
	self.connectionReset();
}
```

## writeItems

```js
NodeS7.prototype.writeItems = function(arg, value, cb) {
	var self = this, i;
	outputLog("Preparing to WRITE " + arg + " to value " + value, 0, self.connectionID);
	if (self.isWriting() || self.writeInQueue) {
		outputLog("Ви повинні зачекати, доки завершаться всі попередні записи," + 
                  "перш ніж запланувати інший." , 0, self.connectionID);
		return 1;  // Слідкуйте за цим у своєму коді - 
        // 1 означає, що він фактично не входив до черги.
	}

	if (typeof cb === "function") {
		self.writeDoneCallback = cb;
	} else {
		self.writeDoneCallback = doNothing;
	}

	self.instantWriteBlockList = []; // Ініціалізація масиву.

	if (typeof arg === "string") {
		self.instantWriteBlockList.push(stringToS7Addr(self.translationCB(arg), 
                                                       arg, self.connectionParams));
		if (typeof (self.instantWriteBlockList[self.instantWriteBlockList.length - 1])
            !== "undefined") {
			self.instantWriteBlockList[self.instantWriteBlockList.length - 1]
                .writeValue = value;
		}
	} else if (Array.isArray(arg) && Array.isArray(value) 
               && (arg.length == value.length)) {
		for (i = 0; i < arg.length; i++) {
			if (typeof arg[i] === "string") {
				self.instantWriteBlockList.push(
                    stringToS7Addr(self.translationCB(arg[i]), arg[i],
                                   self.connectionParams));
				if (typeof (self.instantWriteBlockList[
                    self.instantWriteBlockList.length - 1]) !== "undefined") {
					self.instantWriteBlockList[self.instantWriteBlockList.length 
                                               - 1].writeValue = value[i];
				}
			}
		}
	}

	// Validity check.
	for (i = self.instantWriteBlockList.length - 1; i >= 0; i--) {
		if (self.instantWriteBlockList[i] === undefined) {
			self.instantWriteBlockList.splice(i, 1);
			outputLog("Скидання невизначеного елемента запису.");
		}
	}
	self.prepareWritePacket();
	if (!self.isReading()) {
		self.sendWritePacket();
	} else {
		if (self.writeInQueue) {
			outputLog("Запис уже був у черзі - слід запобігти вище",
                      1,self.connectionID);
		}
		self.writeInQueue = true;
		outputLog("Додавання запису в чергу");
	}
	return 0;
}
```

## readAllItems

```js
NodeS7.prototype.readAllItems = function(arg) {
	var self = this;
	outputLog("Читання всіх елементів (викликано readAllItems)", 1, self.connectionID);
	if (typeof arg === "function") {
		self.readDoneCallback = arg;
	} else {
		self.readDoneCallback = doNothing;
	}

	if (self.isoConnectionState !== 4) {
		outputLog("Неможливо прочитати без підключення. Повернути неправильні значення."
                  , 0, self.connectionID);
	} // Для кращої роботи під час автоматичного повторного підключення - не повертати зараз 

	// Перевірте, чи ВСЕ виконано... Ви можете подумати, 
    //що ми можемо розглянути паралельні завдання, 
    //і здебільшого ми можемо, але якщо один щойно закінчив, і ми опиняємось тут 
    //перш ніж почати іншу, це погано. 
	if (self.isWaiting()) {
		outputLog("Очікування читання для завершення всіх операцій R/W. " + 
                  "Повторно запустить readAllItems через 100 мс.", 0, self.connectionID);
		clearTimeout(self.rereadTimer);
		self.rereadTimer = setTimeout(function() {
			self.rereadTimer = undefined; //вже вистрілили, можете сміливо викидати
			self.readAllItems.apply(self, arguments);
		}, 100, arg);
		return;
	}

	// Now we check the array of adding and removing things.  
    // Only now is it really safe to do this.
	self.addRemoveArray.forEach(function(element) {
		outputLog('Adding or Removing ' + util.format(element), 1, self.connectionID);
		if (element.action === 'remove') {
			self.removeItemsNow(element.arg);
		}
		if (element.action === 'add') {
			self.addItemsNow(element.arg);
		}
	});

	self.addRemoveArray = []; // Clear for next time.

	if (!self.readPacketValid) { self.prepareReadPacket(); }

	// ideally...  incrementSequenceNumbers();

	outputLog("Calling SRP from RAI", 1, self.connectionID);
    // Note this sends the first few read packets depending 
    // on parallel connection restrictions.
	self.sendReadPacket(); 
}
```

## findItem

```js
NodeS7.prototype.findItem = function(useraddr) {
	var self = this, i;
	var commstate = { value: self.isoConnectionState !== 4, quality: 'OK' };
	if (useraddr === '_COMMERR') { return commstate; }
	for (i = 0; i < self.polledReadBlockList.length; i++) {
		if (self.polledReadBlockList[i].useraddr === useraddr) 
        { return self.polledReadBlockList[i]; }
	}
	return undefined;
}
```

## add, remove Item

```js
NodeS7.prototype.addItems = function(arg) {
	var self = this;
	self.addRemoveArray.push({ arg: arg, action: 'add' });
}
```

```js
NodeS7.prototype.addItemsNow = function(arg) {
	var self = this, i;
	outputLog("Adding " + arg, 0, self.connectionID);
	if (typeof (arg) === "string" && arg !== "_COMMERR") {
		self.polledReadBlockList.push(stringToS7Addr(
            self.translationCB(arg), arg, self.connectionParams));
	} else if (Array.isArray(arg)) {
		for (i = 0; i < arg.length; i++) {
			if (typeof (arg[i]) === "string" && arg[i] !== "_COMMERR") {
				self.polledReadBlockList.push(
                    stringToS7Addr(self.translationCB(arg[i]), 
                                   arg[i], self.connectionParams));
			}
		}
	}

	// Validity check.
	for (i = self.polledReadBlockList.length - 1; i >= 0; i--) {
		if (self.polledReadBlockList[i] === undefined) {
			self.polledReadBlockList.splice(i, 1);
			outputLog("Dropping an undefined request item.", 0, self.connectionID);
		}
	}
	//	self.prepareReadPacket();
	self.readPacketValid = false;
}
```

```js
NodeS7.prototype.removeItems = function(arg) {
	var self = this;
	self.addRemoveArray.push({ arg: arg, action: 'remove' });
}
```

```js
NodeS7.prototype.removeItemsNow = function(arg) {
	var self = this, i;
	if (typeof arg === "undefined") {
		self.polledReadBlockList = [];
	} else if (typeof arg === "string") {
		for (i = 0; i < self.polledReadBlockList.length; i++) {
			outputLog('TCBA ' + self.translationCB(arg));
			if (self.polledReadBlockList[i].addr === self.translationCB(arg)) {
				outputLog('Splicing');
				self.polledReadBlockList.splice(i, 1);
			}
		}
	} else if (Array.isArray(arg)) {
		for (i = 0; i < self.polledReadBlockList.length; i++) {
			for (var j = 0; j < arg.length; j++) {
				if (self.polledReadBlockList[i].addr === self.translationCB(arg[j])) {
					self.polledReadBlockList.splice(i, 1);
				}
			}
		}
	}
	self.readPacketValid = false;
	//	self.prepareReadPacket();
}
```

## isReading/isWriting

```js
NodeS7.prototype.isReading = function() {
	var self = this, i;
	// Walk through the array and if any packets are marked as sent, it means we haven't received our final confirmation.
	for (i = 0; i < self.readPacketArray.length; i++) {
		if (self.readPacketArray[i].sent === true) { return true }
	}
	return false;
}

NodeS7.prototype.isWriting = function() {
	var self = this, i;
	// Walk through the array and if any packets are marked as sent, it means we haven't received our final confirmation.
	for (i = 0; i < self.writePacketArray.length; i++) {
		if (self.writePacketArray[i].sent === true) { return true }
	}
	return false;
}
```

## processS7ReadItem

```js
function processS7ReadItem(theItem) {

	var thePointer = 0;
	var strlen = 0;
	var tempString = '';

	if (theItem.arrayLength > 1) {
		// Array value.
		if (theItem.datatype != 'C' && theItem.datatype != 'CHAR') {
			theItem.value = [];
			theItem.quality = [];
		} else {
			theItem.value = '';
			theItem.quality = '';
		}
		var bitShiftAmount = theItem.bitOffset;
		for (var arrayIndex = 0; arrayIndex < theItem.arrayLength; arrayIndex++) {
			if (theItem.qualityBuffer[thePointer] !== 0xC0) {
				if (theItem.quality instanceof Array) {
					theItem.value.push(theItem.badValue());
					theItem.quality.push('BAD ' + theItem.qualityBuffer[thePointer]);
				} else {
					theItem.value = theItem.badValue();
					theItem.quality = 'BAD ' + theItem.qualityBuffer[thePointer];
				}
			} else {
				// If we're a string, quality is not an array.
				if (theItem.quality instanceof Array) {
					theItem.quality.push('OK');
				} else {
					theItem.quality = 'OK';
				}
				switch (theItem.datatype) {

					case "DT":
						theItem.value.push(readDT(theItem.byteBuffer, thePointer, false));
						break;
					case "DTZ":
						theItem.value.push(readDT(theItem.byteBuffer, thePointer, true));
						break;
					case "DTL":
						theItem.value.push(readDTL(theItem.byteBuffer, thePointer, false));
						break;
					case "DTLZ":
						theItem.value.push(readDTL(theItem.byteBuffer, thePointer, true));
						break;
					case "REAL":
						theItem.value.push(theItem.byteBuffer.readFloatBE(thePointer));
						break;
					case "LREAL":
						theItem.value.push(theItem.byteBuffer.readDoubleBE(thePointer));
						break;
					case "LINT":
//						theItem.value.push(theItem.byteBuffer.readBigInt64BE(thePointer));
						break;
					case "DWORD":
						theItem.value.push(theItem.byteBuffer.readUInt32BE(thePointer));
						break;
					case "DINT":
						theItem.value.push(theItem.byteBuffer.readInt32BE(thePointer));
						break;
					case "INT":
						theItem.value.push(theItem.byteBuffer.readInt16BE(thePointer));
						break;
					case "WORD":
						theItem.value.push(theItem.byteBuffer.readUInt16BE(thePointer));
						break;
					case "X":
						theItem.value.push(((theItem.byteBuffer.readUInt8(thePointer) >> (bitShiftAmount)) & 1) ? true : false);
						break;
					case "B":
					case "BYTE":
						theItem.value.push(theItem.byteBuffer.readUInt8(thePointer));
						break;
					case "S":
					case "STRING":
						strlen = theItem.byteBuffer.readUInt8(thePointer+1);
						tempString = '';
						for (var charOffset = 2; charOffset < theItem.dtypelen && (charOffset - 2) < strlen; charOffset++) {
							// say strlen = 1 (one-char string) this char is at arrayIndex of 2.
							// Convert to string.
							tempString += String.fromCharCode(theItem.byteBuffer.readUInt8(thePointer+charOffset));
						}
						theItem.value.push(tempString);
						break;
					case "C":
					case "CHAR":
						// Convert to string.
						theItem.value += String.fromCharCode(theItem.byteBuffer.readUInt8(thePointer));
						break;
					case "TIMER":
					case "COUNTER":
						theItem.value.push(theItem.byteBuffer.readInt16BE(thePointer));
						break;

					default:
						outputLog("Unknown data type in response - should never happen.  Should have been caught earlier.  " + theItem.datatype);
						return 0;
				}
			}
			if (theItem.datatype == 'X') {
				// For bit arrays, we have to do some tricky math to get the pointer to equal the byte offset.
				// Note that we add the bit offset here for the rare case of an array starting at other than zero.  We either have to
				// drop support for this at the request level or support it here.
				bitShiftAmount++;
				if ((((arrayIndex + theItem.bitOffset + 1) % 8) === 0) || (arrayIndex == theItem.arrayLength - 1)) {
					thePointer += theItem.dtypelen;
					bitShiftAmount = 0;
				}
			} else {
				// Add to the pointer every time.
				thePointer += theItem.dtypelen;
			}
		}
	} else {
		// Single value.
		if (theItem.qualityBuffer[thePointer] !== 0xC0) {
			theItem.value = theItem.badValue();
			theItem.quality = ('BAD ' + theItem.qualityBuffer[thePointer]);
		} else {
			theItem.quality = ('OK');
			switch (theItem.datatype) {

				case "DT":
					theItem.value = readDT(theItem.byteBuffer, thePointer, false);
					break;
				case "DTZ":
					theItem.value = readDT(theItem.byteBuffer, thePointer, true);
					break;
				case "DTL":
					theItem.value = readDTL(theItem.byteBuffer, thePointer, false);
					break;
				case "DTLZ":
					theItem.value = readDTL(theItem.byteBuffer, thePointer, true);
					break;
				case "REAL":
					theItem.value = theItem.byteBuffer.readFloatBE(thePointer);
					break;
				case "LREAL":
					theItem.value = theItem.byteBuffer.readDoubleBE(thePointer);
					break;
				case "LINT":
//					theItem.value = theItem.byteBuffer.readBigInt64BE(thePointer);
					break;
				case "DWORD":
					theItem.value = theItem.byteBuffer.readUInt32BE(thePointer);
					break;
				case "DINT":
					theItem.value = theItem.byteBuffer.readInt32BE(thePointer);
					break;
				case "INT":
					theItem.value = theItem.byteBuffer.readInt16BE(thePointer);
					break;
				case "WORD":
					theItem.value = theItem.byteBuffer.readUInt16BE(thePointer);
					break;
				case "X":
					theItem.value = (((theItem.byteBuffer.readUInt8(thePointer) >> (theItem.bitOffset)) & 1) ? true : false);
					break;
				case "B":
				case "BYTE":
					// No support as of yet for signed 8 bit.  This isn't that common in Siemens.
					theItem.value = theItem.byteBuffer.readUInt8(thePointer);
					break;
				case "S":
				case "STRING":
					strlen = theItem.byteBuffer.readUInt8(thePointer+1);
					theItem.value = '';
					for (var charOffset = 2; charOffset < theItem.dtypelen && (charOffset - 2) < strlen; charOffset++) {
						// say strlen = 1 (one-char string) this char is at arrayIndex of 2.
						// Convert to string.
						theItem.value += String.fromCharCode(theItem.byteBuffer.readUInt8(thePointer+charOffset));
					}
					break;
				case "C":
				case "CHAR":
					// No support as of yet for signed 8 bit.  This isn't that common in Siemens.
					theItem.value = String.fromCharCode(theItem.byteBuffer.readUInt8(thePointer));
					break;
				case "TIMER":
				case "COUNTER":
					theItem.value = theItem.byteBuffer.readInt16BE(thePointer);
					break;
				default:
					outputLog("Unknown data type in response - should never happen.  Should have been caught earlier.  " + theItem.datatype);
					return 0;
			}
		}
		thePointer += theItem.dtypelen;
	}

	if (((thePointer) % 2)) { // Odd number.  With the S7 protocol we only request an even number of bytes.  So there will be a filler byte.
		thePointer += 1;
	}

	//	outputLog("We have an item value of " + theItem.value + " for " + theItem.addr + " and pointer of " + thePointer);
	return thePointer; // Should maybe return a value now???
}
```

## S7Item

```js
function S7Item() { // Object
	// Зберігає оригінальну адресу
	this.addr = undefined;
	this.useraddr = undefined;

	// Перша група — властивості, пов’язані з S7 — лише вони означують адресу. 
	this.addrtype = undefined;
	this.datatype = undefined;
	this.dbNumber = undefined;
	this.bitOffset = undefined;
	this.offset = undefined;
	this.arrayLength = undefined;

	// Ці наступні властивості можуть бути обчислені з наведених вище властивостей 
    // і можуть бути перетворені на функції.
	this.dtypelen = undefined;
	this.areaS7Code = undefined;
	this.byteLength = undefined;
	this.byteLengthWithFill = undefined;

	// Зверніть увагу, що транспортні коди читання та транспортні коди запису 
    // будуть однаковими, за винятком бітів, які читаються як байти, 
    // але записуються як біти 
	this.readTransportCode = undefined;
	this.writeTransportCode = undefined;

	// Саме сюди можуть надходити дані, які надходять у пакеті, 
    // перед обчисленням значення. 
	this.byteBuffer = Buffer.alloc(8192);
	this.writeBuffer = Buffer.alloc(8192);

	// Ми використовуємо "quality buffer", щоб відстежувати, чи були запити успішними.
	// В іншому випадку дуже легко втратити слід масивів, 
    // які можуть бути лише частково повними. 
	this.qualityBuffer = Buffer.alloc(8192);
	this.writeQualityBuffer = Buffer.alloc(8192);

	// Далі у нас є властивості елементів
	this.value = undefined;
	this.writeValue = undefined;
	this.valid = false;
	this.errCode = undefined;

	// Далі ми маємо властивості результату
	this.part = undefined;
	this.maxPart = undefined;

	// Block properties
	this.isOptimized = false;
	this.resultReference = undefined;
	this.itemReference = undefined;

	// And functions...
	this.clone = function() {
		var newObj = new S7Item();
		for (var i in this) {
			if (i == 'clone') continue;
			newObj[i] = this[i];
		} return newObj;
	};

	this.badValue = function() {
		switch (this.datatype) {
			case "DT":
			case "DTZ":
			case "DTL":
			case "DTLZ":
				return new Date(NaN);
			case "REAL":
			case "LREAL":
				return 0.0;
			case "DWORD":
			case "DINT":
			case "INT":
			case "LINT":
			case "WORD":
			case "B":
			case "BYTE":
			case "TIMER":
			case "COUNTER":
				return 0;
			case "X":
				return false;
			case "C":
			case "CHAR":
			case "S":
			case "STRING":
				// Convert to string.
				return "";
			default:
				outputLog("Невідомий тип даних під час визначення неправильного значення" 
                          + " - ніколи не повинно статися. Треба було зловити раніше." +
                          this.datatype);
				return 0;
		}
	};
}
```

## isOptimizableArea

```js
NodeS7.prototype.isOptimizableArea = function(area) {
	var self = this;

	if (self.doNotOptimize) { return false; } // Are we skipping all optimization due to user request?
	switch (area) {
		case 0x84: // db
		case 0x81: // input bytes
		case 0x82: // output bytes
		case 0x83: // memory bytes
			return true;
		default:
			return false;
	}
}
```

## processS7WriteItem

```js
function processS7WriteItem(theData, theItem, thePointer) {

	var remainingLength;

	if (!theData) {
		theItem.writeQualityBuffer.fill(0xFF);  // Note that ff is good in the S7 world but BAD in our fill here.
		theItem.valid = false;
		theItem.errCode = 'We must have timed Out - we have no response to process';
		outputLog(theItem.errCode);
		return 0;
	}

	remainingLength = theData.length - thePointer;  // Say if length is 39 and pointer is 35 we can access 35,36,37,38 = 4 bytes.

	if (remainingLength < 1) {
		theItem.writeQualityBuffer.fill(0xFF);  // Note that ff is good in the S7 world but BAD in our fill here.
		theItem.valid = false;
		theItem.errCode = 'Malformed Packet - Less Than 1 Byte.  TDL ' + theData.length + ' TP' + thePointer + ' RL' + remainingLength;
		outputLog(theItem.errCode);
		return 0;   			// Hard to increment the pointer so we call it a malformed packet and we're done.
	}

	var writeResponse = theData.readUInt8(thePointer);

	theItem.writeResponse = writeResponse;

	if (writeResponse !== 0xff) {
		outputLog('Received write error of ' + theItem.writeResponse + ' on ' + theItem.addr);
		theItem.writeQualityBuffer.fill(0xFF);  // Note that ff is good in the S7 world but BAD in our fill here.
	} else {
		theItem.writeQualityBuffer.fill(0xC0);
	}

	return (thePointer + 1);
}
```

## writePostProcess

```js
function writePostProcess(theItem) {
	var thePointer = 0;
	if (theItem.arrayLength === 1) {
		if (theItem.writeQualityBuffer[0] === 0xFF) {
			theItem.writeQuality = 'BAD';
		} else {
			theItem.writeQuality = 'OK';
		}
	} else {
		// Array value.
		theItem.writeQuality = [];
		for (var arrayIndex = 0; arrayIndex < theItem.arrayLength; arrayIndex++) {
			if (theItem.writeQualityBuffer[thePointer] === 0xFF) {
				theItem.writeQuality[arrayIndex] = 'BAD';
			} else {
				theItem.writeQuality[arrayIndex] = 'OK';
			}
			if (theItem.datatype == 'X') {
				// For bit arrays, we have to do some tricky math to get the pointer to equal the byte offset.
				// Note that we add the bit offset here for the rare case of an array starting at other than zero.  We either have to
				// drop support for this at the request level or support it here.

				if ((((arrayIndex + theItem.bitOffset + 1) % 8) === 0) || (arrayIndex == theItem.arrayLength - 1)) {
					thePointer += theItem.dtypelen;
				}
			} else {
				// Add to the pointer every time.
				thePointer += theItem.dtypelen;
			}
		}
	}
}
```

## getWriteBuffer

```js
function getWriteBuffer(theItem) {
	var newBuffer;

	// NOTE: It seems that when writing, the data that is sent must have a "fill byte" so that data length is even only for all
	//  but the last request.  The last request must have no padding.  So we DO NOT add the padding here anymore.

	if (theItem.datatype === 'X' && theItem.arrayLength === 1) {
		newBuffer = Buffer.alloc(2 + 3); // Changed from 2 + 4 to 2 + 3 as padding was moved out of this function
		// Initialize, especially be sure to get last bit which may be a fill bit.
		newBuffer.fill(0);
		newBuffer.writeUInt16BE(1, 2); // Might need to do something different for different trans codes
	} else {
		newBuffer = Buffer.alloc(theItem.byteLength + 4); // Changed from 2 + 4 to 2 + 3 as padding was moved out of this function
		newBuffer.fill(0);
		newBuffer.writeUInt16BE(theItem.byteLength * 8, 2); // Might need to do something different for different trans codes
	}

	if (theItem.writeBuffer.length < theItem.byteLengthWithFill) {
		outputLog("Attempted to access part of the write buffer that wasn't there when writing an item.");
	}

	newBuffer[0] = 0;
	newBuffer[1] = theItem.writeTransportCode;

	theItem.writeBuffer.copy(newBuffer, 4, 0, theItem.byteLength);  // Not with fill.  It might not be that long.

	return newBuffer;
}
```

## bufferizeS7Item

```js
function bufferizeS7Item(theItem) {
	var thePointer, theByte;
	theByte = 0;
	thePointer = 0; // After length and header

	if (theItem.arrayLength > 1) {
		// Array value.
		var bitShiftAmount = theItem.bitOffset;
		for (var arrayIndex = 0; arrayIndex < theItem.arrayLength; arrayIndex++) {
			switch (theItem.datatype) {
				case "DT":
					writeDT(theItem.writeValue[arrayIndex], theItem.writeBuffer, thePointer, false);
					break;
				case "DTZ":
					writeDT(theItem.writeValue[arrayIndex], theItem.writeBuffer, thePointer, true);
					break;
				case "DTL":
					writeDTL(theItem.writeValue[arrayIndex], theItem.writeBuffer, thePointer, false);
					break;
				case "DTLZ":
					writeDTL(theItem.writeValue[arrayIndex], theItem.writeBuffer, thePointer, true);
					break;
				case "REAL":
					theItem.writeBuffer.writeFloatBE(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "LREAL":
					theItem.writeBuffer.writeDoubleBE(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "LINT":
//					theItem.writeBuffer.writeBigInt64BE(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "DWORD":
					theItem.writeBuffer.writeInt32BE(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "DINT":
					theItem.writeBuffer.writeInt32BE(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "INT":
					theItem.writeBuffer.writeInt16BE(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "WORD":
					theItem.writeBuffer.writeUInt16BE(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "X":
					theByte = theByte | (((theItem.writeValue[arrayIndex] === true) ? 1 : 0) << bitShiftAmount);
					// Maybe not so efficient to do this every time when we only need to do it every 8.  Need to be careful with optimizations here for odd requests.
					theItem.writeBuffer.writeUInt8(theByte, thePointer);
					bitShiftAmount++;
					break;
				case "B":
				case "BYTE":
					theItem.writeBuffer.writeUInt8(theItem.writeValue[arrayIndex], thePointer);
					break;
				case "C":
				case "CHAR":
					// Convert to string.
					//??					theItem.writeBuffer.writeUInt8(theItem.writeValue.toCharCode(), thePointer);
					theItem.writeBuffer.writeUInt8(theItem.writeValue.charCodeAt(arrayIndex), thePointer);
					break;
				case "S":
				case "STRING":
					// Convert to bytes.
					theItem.writeBuffer.writeUInt8(theItem.dtypelen - 2, thePointer); // Array length is requested val, -2 is string length
					theItem.writeBuffer.writeUInt8(Math.min(theItem.dtypelen - 2, theItem.writeValue[arrayIndex].length), thePointer+1); // Array length is requested val, -2 is string length
					for (var charOffset = 2; charOffset < theItem.dtypelen; charOffset++) {
						if (charOffset < (theItem.writeValue[arrayIndex].length + 2)) {
							theItem.writeBuffer.writeUInt8(theItem.writeValue[arrayIndex].charCodeAt(charOffset-2), thePointer+charOffset);
						} else {
							theItem.writeBuffer.writeUInt8(32, thePointer+charOffset); // write space
						}
					}
					break;
				case "TIMER":
				case "COUNTER":
					// I didn't think we supported arrays of timers and counters.
					theItem.writeBuffer.writeInt16BE(theItem.writeValue[arrayIndex], thePointer);
					break;
				default:
					outputLog("Unknown data type when preparing array write packet - should never happen.  Should have been caught earlier.  " + theItem.datatype);
					return 0;
			}
			if (theItem.datatype == 'X') {
				// For bit arrays, we have to do some tricky math to get the pointer to equal the byte offset.
				// Note that we add the bit offset here for the rare case of an array starting at other than zero.  We either have to
				// drop support for this at the request level or support it here.
				if ((((arrayIndex + theItem.bitOffset + 1) % 8) === 0) || (arrayIndex == theItem.arrayLength - 1)) {
					thePointer += theItem.dtypelen;
					bitShiftAmount = 0;
					// Zero this now.  Otherwise it will have the same value next byte if non-zero.
					theByte = 0;
				}
			} else {
				// Add to the pointer every time.
				thePointer += theItem.dtypelen;
			}
		}
	} else {
		// Single value.
		switch (theItem.datatype) {

			case "DT":
				writeDT(theItem.writeValue, theItem.writeBuffer, thePointer, false);
				break;
			case "DTZ":
				writeDT(theItem.writeValue, theItem.writeBuffer, thePointer, true);
				break;
			case "DTL":
				writeDTL(theItem.writeValue, theItem.writeBuffer, thePointer, false);
				break;
			case "DTLZ":
				writeDTL(theItem.writeValue, theItem.writeBuffer, thePointer, true);
				break;
			case "REAL":
				theItem.writeBuffer.writeFloatBE(theItem.writeValue, thePointer);
				break;
			case "LREAL":
				theItem.writeBuffer.writeDoubleBE(theItem.writeValue, thePointer);
				break;
			case "LINT":
//				theItem.writeBuffer.writeBigInt64BE(theItem.writeValue, thePointer);
				break;
			case "DWORD":
				theItem.writeBuffer.writeUInt32BE(theItem.writeValue, thePointer);
				break;
			case "DINT":
				theItem.writeBuffer.writeInt32BE(theItem.writeValue, thePointer);
				break;
			case "INT":
				theItem.writeBuffer.writeInt16BE(theItem.writeValue, thePointer);
				break;
			case "WORD":
				theItem.writeBuffer.writeUInt16BE(theItem.writeValue, thePointer);
				break;
			case "X":
				theItem.writeBuffer.writeUInt8(((theItem.writeValue === true) ? 1 : 0), thePointer);
				// not here				theItem.writeBuffer[1] = 1; // Set transport code to "BIT" to write a single bit.
				// not here				theItem.writeBuffer.writeUInt16BE(1, 2); // Write only one bit.
				break;
			case "B":
			case "BYTE":
				// No support as of yet for signed 8 bit.  This isn't that common in Siemens.
				theItem.writeBuffer.writeUInt8(theItem.writeValue, thePointer);
				break;
			case "C":
			case "CHAR":
				// No support as of yet for signed 8 bit.  This isn't that common in Siemens.
				theItem.writeBuffer.writeUInt8(theItem.writeValue.charCodeAt(0), thePointer);
				break;
			case "S":
			case "STRING":
				// Convert to bytes.
				theItem.writeBuffer.writeUInt8(theItem.dtypelen - 2, thePointer); // Array length is requested val, -2 is string length
				theItem.writeBuffer.writeUInt8(Math.min(theItem.dtypelen - 2, theItem.writeValue.length), thePointer+1); // Array length is requested val, -2 is string length

				for (var charOffset = 2; charOffset < theItem.dtypelen; charOffset++) {
					if (charOffset < (theItem.writeValue.length + 2)) {
						theItem.writeBuffer.writeUInt8(theItem.writeValue.charCodeAt(charOffset-2), thePointer+charOffset);
					} else {
						theItem.writeBuffer.writeUInt8(32, thePointer+charOffset); // write space
					}
				}
				break;
			case "TIMER":
			case "COUNTER":
				theItem.writeBuffer.writeInt16BE(theItem.writeValue, thePointer);
				break;
			default:
				outputLog("Unknown data type in write prepare - should never happen.  Should have been caught earlier.  " + theItem.datatype);
				return 0;
		}
		thePointer += theItem.dtypelen;
	}
	return undefined;
}
```

## stringToS7Addr

```js
function stringToS7Addr(addr, useraddr, cParam) {
	"use strict";
	var theItem, splitString, splitString2;
	// Спеціальний випадок для статусу помилки зв’язку – ця змінна 
    // повертає значення true, коли виникає помилка зв’язку
	if (useraddr === '_COMMERR') { return undefined; } 

	theItem = new S7Item();
	splitString = addr.split(',');
	if (splitString.length === 0 || splitString.length > 2) {
		outputLog("Помилка - Рядок не вдалося розділити належним чином.");
		return undefined;
	}

	if (splitString.length > 1) { // Має бути типу DB, напиклад 'DB1,REAL8'
		theItem.addrtype = 'DB';  // Hard code
		splitString2 = splitString[1].split('.');
        // Убираємо числа
		theItem.datatype = splitString2[0].replace(/[0-9]/gi, '').toUpperCase(); 
		if (theItem.datatype === 'X' && splitString2.length === 3) {
			theItem.arrayLength = parseInt(splitString2[2], 10);
		} else if ((theItem.datatype === 'S' || theItem.datatype === 'STRING') 
                   && splitString2.length === 3) {
            // З рядками додайте 2 до довжини через заголовок S7
			theItem.dtypelen = parseInt(splitString2[1], 10) + 2;
            // Для рядків довжина масиву тепер дорівнює кількості рядків
			theItem.arrayLength = parseInt(splitString2[2], 10);  
		} else if ((theItem.datatype === 'S' || theItem.datatype === 'STRING') 
                   && splitString2.length === 2) {
            // З рядками додайте 2 до довжини через заголовок S7
			theItem.dtypelen = parseInt(splitString2[1], 10) + 2;
			theItem.arrayLength = 1;
		} else if (theItem.datatype !== 'X' && splitString2.length === 2) {
			theItem.arrayLength = parseInt(splitString2[1], 10);
		} else {
			theItem.arrayLength = 1;
		}
		if (theItem.arrayLength <= 0) {
			outputLog('Zero length arrays not allowed, returning undefined');
			return undefined;
		}

		// Отримайте номер блоку даних з першої частини.
		theItem.dbNumber = parseInt(splitString[0].replace(/[A-z]/gi, ''), 10);

		// Отримайте байтове зміщення блоку даних від другої частини, виключаючи символи.
        // Зауважте, що на цьому етапі ми можемо пропустити деяку інформацію, 
        // як-от літеру "T" у кінці, яка вказує на тип даних TIME, 
        // тип даних DATE або тип даних DT. Ми їх ігноруємо.
		// Це в списку TODO.
        // Позбутися символів
		theItem.offset = parseInt(splitString2[0].replace(/[A-z]/gi, ''), 10);  

		// Отримати бітове зміщення
		if (splitString2.length > 1 && theItem.datatype === 'X') {
			theItem.bitOffset = parseInt(splitString2[1], 10);
			if (theItem.bitOffset > 7) {
				outputLog("Invalid bit offset specified for address " + addr);
				return undefined;
			}
		}
	} else { // Не має бути DB. Ми знаємо, що коми немає. 
		splitString2 = addr.split('.');

		switch (splitString2[0].replace(/[0-9]/gi, '')) {
			/* We do have the memory areas:
			  "input", "peripheral input", "output", 
			  "peripheral output", ",marker", "counter" 
			  and "timer" as I, PI, Q, PQ, M, C and T.
			   Datablocks are handles somewere else.
			   We do have the data types:
			   "bit", "byte", "char", "word", "int16", "dword", 
			   "int32", "real" as X, B, C, W, I, DW, DI and R
			   What about "uint16", "uint32"
			*/

		/* Усі стилі периферійних операцій вводу-виводу (доступ до бітів заборонений) */
			case "PIB":
			case "PEB":
			case "PQB":
			case "PAB":
				theItem.addrtype = "P";
				theItem.datatype = "BYTE";
				break;
			case "PIC":
			case "PEC":
			case "PQC":
			case "PAC":
				theItem.addrtype = "P";
				theItem.datatype = "CHAR";
				break;
			case "PIW":
			case "PEW":
			case "PQW":
			case "PAW":
				theItem.addrtype = "P";
				theItem.datatype = "WORD";
				break;
			case "PII":
			case "PEI":
			case "PQI":
			case "PAI":
				theItem.addrtype = "P";
				theItem.datatype = "INT";
				break;
			case "PID":
			case "PED":
			case "PQD":
			case "PAD":
				theItem.addrtype = "P";
				theItem.datatype = "DWORD";
				break;
			case "PIDI":
			case "PEDI":
			case "PQDI":
			case "PADI":
				theItem.addrtype = "P";
				theItem.datatype = "DINT";
				break;
			case "PIR":
			case "PER":
			case "PQR":
			case "PAR":
				theItem.addrtype = "P";
				theItem.datatype = "REAL";
				break;

/* Усі стилі стандартних входів (на відміну від периферійних входів)  */
			case "I":
			case "E":
				theItem.addrtype = "I";
				theItem.datatype = "X";
				break;
			case "IB":
			case "EB":
				theItem.addrtype = "I";
				theItem.datatype = "BYTE";
				break;
			case "IC":
			case "EC":
				theItem.addrtype = "I";
				theItem.datatype = "CHAR";
				break;
			case "IW":
			case "EW":
				theItem.addrtype = "I";
				theItem.datatype = "WORD";
				break;
			case "II":
			case "EI":
				theItem.addrtype = "I";
				theItem.datatype = "INT";
				break;
			case "ID":
			case "ED":
				theItem.addrtype = "I";
				theItem.datatype = "DWORD";
				break;
			case "IDI":
			case "EDI":
				theItem.addrtype = "I";
				theItem.datatype = "DINT";
				break;
			case "IR":
			case "ER":
				theItem.addrtype = "I";
				theItem.datatype = "REAL";
				break;
			case "ILR":
			case "ELR":
				theItem.addrtype = "I";
				theItem.datatype = "LREAL";
				break;
			case "ILI":
			case "ELI":
				theItem.addrtype = "I";
				theItem.datatype = "LINT";
				break;
/* All styles of standard outputs (in oposit to peripheral outputs) */
			case "Q":
			case "A":
				theItem.addrtype = "Q";
				theItem.datatype = "X";
				break;
			case "QB":
			case "AB":
				theItem.addrtype = "Q";
				theItem.datatype = "BYTE";
				break;
			case "QC":
			case "AC":
				theItem.addrtype = "Q";
				theItem.datatype = "CHAR";
				break;
			case "QW":
			case "AW":
				theItem.addrtype = "Q";
				theItem.datatype = "WORD";
				break;
			case "QI":
			case "AI":
				theItem.addrtype = "Q";
				theItem.datatype = "INT";
				break;
			case "QD":
			case "AD":
				theItem.addrtype = "Q";
				theItem.datatype = "DWORD";
				break;
			case "QDI":
			case "ADI":
				theItem.addrtype = "Q";
				theItem.datatype = "DINT";
				break;
			case "QR":
			case "AR":
				theItem.addrtype = "Q";
				theItem.datatype = "REAL";
				break;
			case "QLR":
			case "ALR":
				theItem.addrtype = "Q";
				theItem.datatype = "LREAL";
				break;
			case "QLI":
			case "ALI":
				theItem.addrtype = "Q";
				theItem.datatype = "LINT";
				break;
/* Всі стилі маркера */
			case "M":
				theItem.addrtype = "M";
				theItem.datatype = "X";
				break;
			case "MB":
				theItem.addrtype = "M";
				theItem.datatype = "BYTE";
				break;
			case "MC":
				theItem.addrtype = "M";
				theItem.datatype = "CHAR";
				break;
			case "MW":
				theItem.addrtype = "M";
				theItem.datatype = "WORD";
				break;
			case "MI":
				theItem.addrtype = "M";
				theItem.datatype = "INT";
				break;
			case "MD":
				theItem.addrtype = "M";
				theItem.datatype = "DWORD";
				break;
			case "MDI":
				theItem.addrtype = "M";
				theItem.datatype = "DINT";
				break;
			case "MR":
				theItem.addrtype = "M";
				theItem.datatype = "REAL";
				break;
			case "MLR":
				theItem.addrtype = "M";
				theItem.datatype = "LREAL";
				break;
			case "MLI":
				theItem.addrtype = "M";
				theItem.datatype = "LINT";
				break;
/* Timer */
			case "T":
				theItem.addrtype = "T";
				theItem.datatype = "TIMER";
				break;

/* Counter */
			case "C":
				theItem.addrtype = "C";
				theItem.datatype = "COUNTER";
				break;

			default:
				outputLog('Failed to find a match for ' + splitString2[0]);
				return undefined;
		}

		theItem.bitOffset = 0;
		if (splitString2.length > 1 && theItem.datatype === 'X') { // Bit and bit array
			theItem.bitOffset = parseInt(splitString2[1].replace(/[A-z]/gi, ''), 10);
			if (splitString2.length > 2) {  // Bit array only
				theItem.arrayLength = parseInt(splitString2[2].replace(/[A-z]/gi, '')
                                               , 10);
			} else {
				theItem.arrayLength = 1;
			}
		} // Bit and bit array
        else if (splitString2.length > 1 && theItem.datatype !== 'X') { 
			theItem.arrayLength = parseInt(splitString2[1].replace(/[A-z]/gi, ''), 10);
		} else {
			theItem.arrayLength = 1;
		}
		theItem.dbNumber = 0;
		theItem.offset = parseInt(splitString2[0].replace(/[A-z]/gi, ''), 10);
	}

	if (theItem.datatype === 'DI') {
		theItem.datatype = 'DINT';
	}
	if (theItem.datatype === 'I') {
		theItem.datatype = 'INT';
	}
	if (theItem.datatype === 'DW' || theItem.datatype === 'DWT') {
		theItem.datatype = 'DWORD';
	}
	if (theItem.datatype === 'WDT') {
		if (cParam.wdtAsUTC) {
			theItem.datatype = 'DTZ';
		} else {
			theItem.datatype = 'DT';
		}
	}
	if (theItem.datatype === 'W') {
		theItem.datatype = 'WORD';
	}
	if (theItem.datatype === 'R') {
		theItem.datatype = 'REAL';
	}
	if (theItem.datatype === 'LR') {
		theItem.datatype = 'LREAL';
	}
	if (theItem.datatype === 'LI') {
		theItem.datatype = 'LINT';
	}
	switch (theItem.datatype) {
		case "DTL":
		case "DTLZ":
			theItem.dtypelen = 12;
			break;
		case "LREAL":
		case "LINT":
		case "DT":
		case "DTZ":
			theItem.dtypelen = 8;
			break;
		case "REAL":
		case "DWORD":
		case "DINT":
			theItem.dtypelen = 4;
			break;
		case "INT":
		case "WORD":
		case "TIMER":
		case "COUNTER":
			theItem.dtypelen = 2;
			break;
		case "X":
		case "B":
		case "C":
		case "BYTE":
		case "CHAR":
			theItem.dtypelen = 1;
			break;
		case "S":
		case "STRING":
			// For strings, arrayLength and dtypelen were assigned during parsing.
			break;
		default:
			outputLog("Unknown data type " + theItem.datatype);
			return undefined;
	}

	// Default
	theItem.readTransportCode = 0x04;

	switch (theItem.addrtype) {
		case "DB":
		case "DI":
			theItem.areaS7Code = 0x84;
			break;
		case "I":
		case "E":
			theItem.areaS7Code = 0x81;
			break;
		case "Q":
		case "A":
			theItem.areaS7Code = 0x82;
			break;
		case "M":
			theItem.areaS7Code = 0x83;
			break;
		case "P":
			theItem.areaS7Code = 0x80;
			break;
		case "C":
			theItem.areaS7Code = 0x1c;
			theItem.readTransportCode = 0x09;
			break;
		case "T":
			theItem.areaS7Code = 0x1d;
			theItem.readTransportCode = 0x09;
			break;
		default:
			outputLog("Unknown memory area entered - " + theItem.addrtype);
			return undefined;
	}

	if (theItem.datatype === 'X' && theItem.arrayLength === 1) {
		theItem.writeTransportCode = 0x03;
	} else {
		theItem.writeTransportCode = theItem.readTransportCode;
	}

	// Save the address from the argument for later use and reference
	theItem.addr = addr;
	if (useraddr === undefined) {
		theItem.useraddr = addr;
	} else {
		theItem.useraddr = useraddr;
	}

	if (theItem.datatype === 'X') {
		theItem.byteLength = Math.ceil((theItem.bitOffset + theItem.arrayLength) / 8);
	} else {
		theItem.byteLength = theItem.arrayLength * theItem.dtypelen;
	}

	theItem.byteLengthWithFill = theItem.byteLength;
    // S7 додасть байт-заповнювач. 
    // Використовуйте цю очікувану довжину відповіді для обчислень PDU.
	if (theItem.byteLengthWithFill % 2) { theItem.byteLengthWithFill += 1; }  

	return theItem;
}
```

## S7AddrToBuffer

```js
function S7AddrToBuffer(addrinfo, isWriting) {
	var thisBitOffset = 0, theReq = Buffer.from([0x12, 0x0a, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]);

	// First 3 bytes (0,1,2) is constant, sniffed from other traffic, for S7 head.
	// Next one is "byte length" - we always request X number of bytes - even for a REAL with length of 1 we read BYTES length of 4.
	theReq[3] = 0x02;  // Byte length

	// Next we write the number of bytes we are going to read.
	if (addrinfo.datatype === 'X') {
		theReq.writeUInt16BE(addrinfo.byteLength, 4);
		if (isWriting && addrinfo.arrayLength === 1) {
			// Byte length will be 1 already so no need to special case this.
			theReq[3] = 0x01;  // 1 = "BIT" length
			// We need to specify the bit offset in this case only.  Normally, when reading, we read the whole byte anyway and shift bits around.  Can't do this when writing only one bit.
			thisBitOffset = addrinfo.bitOffset;
		}
	} else if (addrinfo.datatype === 'TIMER' || addrinfo.datatype === 'COUNTER') {
		theReq.writeUInt16BE(1, 4);
		theReq.writeUInt8(addrinfo.areaS7Code, 3);
	} else {
		theReq.writeUInt16BE(addrinfo.byteLength, 4);
	}

	// Then we write the data block number.
	theReq.writeUInt16BE(addrinfo.dbNumber, 6);

	// Write our area crossing pointer.  When reading, write a bit offset of 0 - we shift the bit offset out later only when reading.
	theReq.writeUInt32BE(addrinfo.offset * 8 + thisBitOffset, 8);

	// Now we have to BITWISE OR the area code over the area crossing pointer.
	// This must be done AFTER writing the area crossing pointer as there is overlap, but this will only be noticed on large DB.
	theReq[8] |= addrinfo.areaS7Code;

	return theReq;
}
```

## setTranslationCB

```js
NodeS7.prototype.setTranslationCB = function(cb) {
	var self = this;
	if (typeof cb === "function") {
		outputLog('Translation OK');
		self.translationCB = cb;
	}
}
```

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

## BCD

```js
function fromBCD(n) {
	return ((n >> 4) * 10) + (n & 0xf)
}

function toBCD(n) {
	return ((n / 10) << 4) | (n % 10)
}
```

## DT/DTL

```js
function readDT(buffer, offset, isUTC) {
	let year = fromBCD(buffer.readUInt8(offset));
	let month = fromBCD(buffer.readUInt8(offset + 1));
	let day = fromBCD(buffer.readUInt8(offset + 2));
	let hour = fromBCD(buffer.readUInt8(offset + 3));
	let min = fromBCD(buffer.readUInt8(offset + 4));
	let sec = fromBCD(buffer.readUInt8(offset + 5));
	let ms_1 = fromBCD(buffer.readUInt8(offset + 6));
	let ms_2 = fromBCD(buffer.readUInt8(offset + 7) & 0xf0);

	let date;
	if (isUTC) {
		date = new Date(Date.UTC((year > 89 ? 1900 : 2000) + year, month - 1,
			day, hour, min, sec, (ms_1 * 10) + (ms_2 / 10)))
	} else {
		date = new Date((year > 89 ? 1900 : 2000) + year, month - 1,
			day, hour, min, sec, (ms_1 * 10) + (ms_2 / 10));
	}

	return date;
}

function writeDT(date, buffer, offset, isUTC){
	if (!(date instanceof Date)) {
		if (date > 631152000000 && date < 3786911999999) {
			// is between "1990-01-01T00:00:00.000Z" and "2089-12-31T23:59:59.999Z" in JS epoch
			// as per data type's range definition
			date = new Date(date);
		} else {
			outputLog("Unsupported value of [" + date + "] for writing data of type DATE_AND_TIME. Skipping item");
			return;
		}
	}

	if (isUTC) {
		buffer.writeUInt8(toBCD(date.getUTCFullYear() % 100), offset);
		buffer.writeUInt8(toBCD(date.getUTCMonth() + 1), offset + 1);
		buffer.writeUInt8(toBCD(date.getUTCDate()), offset + 2);
		buffer.writeUInt8(toBCD(date.getUTCHours()), offset + 3);
		buffer.writeUInt8(toBCD(date.getUTCMinutes()), offset + 4);
		buffer.writeUInt8(toBCD(date.getUTCSeconds()), offset + 5);
		buffer.writeUInt8(toBCD((date.getUTCMilliseconds() / 10) >> 0), offset + 6);
		buffer.writeUInt8(toBCD(((date.getUTCMilliseconds() % 10) * 10) + (date.getUTCDay() + 1)), offset + 7);
	} else {
		buffer.writeUInt8(toBCD(date.getFullYear() % 100), offset);
		buffer.writeUInt8(toBCD(date.getMonth() + 1), offset + 1);
		buffer.writeUInt8(toBCD(date.getDate()), offset + 2);
		buffer.writeUInt8(toBCD(date.getHours()), offset + 3);
		buffer.writeUInt8(toBCD(date.getMinutes()), offset + 4);
		buffer.writeUInt8(toBCD(date.getSeconds()), offset + 5);
		buffer.writeUInt8(toBCD((date.getMilliseconds() / 10) >> 0), offset + 6);
		buffer.writeUInt8(toBCD(((date.getMilliseconds() % 10) * 10) + (date.getDay() + 1)), offset + 7);
	}
}

function readDTL(buffer, offset, isUTC) {
	let year = buffer.readUInt16BE(offset);
	let month = buffer.readUInt8(offset + 2);
	let day = buffer.readUInt8(offset + 3);
	//let weekday = buffer.readUInt8(offset + 4);
	let hour = buffer.readUInt8(offset + 5);
	let min = buffer.readUInt8(offset + 6);
	let sec = buffer.readUInt8(offset + 7);
	let ns = buffer.readUInt32BE(offset + 8);

	let date;
	if (isUTC) {
		date = new Date(Date.UTC(year, month - 1,
			day, hour, min, sec, ns / 1e6))
	} else {
		date = new Date(year, month - 1,
			day, hour, min, sec, ns / 1e6);
	}

	return date;
}

function writeDTL(date, buffer, offset, isUTC) {
	if (!(date instanceof Date)) {
		if (date >= 0 && date < 9223382836854) {
			// is between "1970-01-01T00:00:00.000Z" and "2262-04-11T23:47:16.854Z" in JS epoch
			// as per data type's range definition
			date = new Date(date);
		} else {
			outputLog("Unsupported value of [" + date + "] for writing data of type DATE_AND_TIME. Skipping item");
			return;
		}
	}

	if (isUTC) {
		buffer.writeUInt16BE(date.getUTCFullYear(), offset);
		buffer.writeUInt8(date.getUTCMonth() + 1, offset + 2);
		buffer.writeUInt8(date.getUTCDate(), offset + 3);
		buffer.writeUInt8(date.getUTCDay() + 1, offset + 4);
		buffer.writeUInt8(date.getUTCHours(), offset + 5);
		buffer.writeUInt8(date.getUTCMinutes(), offset + 6);
		buffer.writeUInt8(date.getUTCSeconds(), offset + 7);
		buffer.writeUInt32BE(date.getUTCMilliseconds() * 1e6, offset + 8);
	} else {
		buffer.writeUInt16BE(date.getFullYear(), offset);
		buffer.writeUInt8(date.getMonth() + 1, offset + 2);
		buffer.writeUInt8(date.getDate(), offset + 3);
		buffer.writeUInt8(date.getDay() + 1, offset + 4);
		buffer.writeUInt8(date.getHours(), offset + 5);
		buffer.writeUInt8(date.getMinutes(), offset + 6);
		buffer.writeUInt8(date.getSeconds(), offset + 7);
		buffer.writeUInt32BE(date.getMilliseconds() * 1e6, offset + 8);
	}
}
```

