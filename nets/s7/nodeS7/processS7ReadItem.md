[<-- До опису бібліотеки](README.md) 

## processS7ReadItem

Перетворює theItem:

- виставляє `value` по `byteBuffer` в залежності від `datatype`

```
processS7ReadItem(theItem) 
```



```js
function processS7ReadItem(theItem) {

	var thePointer = 0;
	var strlen = 0;
	var tempString = '';

	if (theItem.arrayLength > 1) {
		// Значення масиву.
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
						theItem.value.push(readDT(theItem.byteBuffer, 
                                                  thePointer, false));
						break;
					case "DTZ":
						theItem.value.push(readDT(theItem.byteBuffer, 
                                                  thePointer, true));
						break;
					case "DTL":
						theItem.value.push(readDTL(theItem.byteBuffer, 
                                                   thePointer, false));
						break;
					case "DTLZ":
						theItem.value.push(readDTL(theItem.byteBuffer, 
                                                   thePointer, true));
						break;
					case "REAL":
						theItem.value.push(theItem.byteBuffer.readFloatBE(thePointer));
						break;
					case "LREAL":
						theItem.value.push(theItem.byteBuffer.readDoubleBE(thePointer));
						break;
					case "LINT":
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
						theItem.value.push(((theItem.byteBuffer.readUInt8(thePointer) 
                                             >> (bitShiftAmount)) & 1) ? true : false);
						break;
					case "B":
					case "BYTE":
						theItem.value.push(theItem.byteBuffer.readUInt8(thePointer));
						break;
					case "S":
					case "STRING":
						strlen = theItem.byteBuffer.readUInt8(thePointer+1);
						tempString = '';
						for (var charOffset = 2; charOffset < theItem.dtypelen 
                             && (charOffset - 2) < strlen; charOffset++) {
							// скажімо, strlen = 1 (рядок з одного символу), 
                            // цей символ має arrayIndex 2. 
							// Перетворити на рядок.
							tempString += String.fromCharCode(
                                theItem.byteBuffer.readUInt8(thePointer+charOffset));
						}
						theItem.value.push(tempString);
						break;
					case "C":
					case "CHAR":
						// Перетворити на рядок.
						theItem.value += String.fromCharCode(
                            theItem.byteBuffer.readUInt8(thePointer));
						break;
					case "TIMER":
					case "COUNTER":
						theItem.value.push(theItem.byteBuffer.readInt16BE(thePointer));
						break;

					default:
						outputLog("Невідомий тип даних у відповідь - ніколи" + 
                                  "не повинно статися. Треба було зловити раніше.  " 
                                  + theItem.datatype);
						return 0;
				}
			}
			if (theItem.datatype == 'X') {
				// Для бітових масивів нам потрібно виконати хитру математику, 
                // щоб змусити вказівник дорівнювати зміщенню байтів.
				// Зауважте, що ми додаємо сюди бітове зміщення для рідкісних випадків,
                // коли масив починається не з нуля. Ми повинні або відмовитися від
                // підтримки цього на рівні запиту, або підтримати це тут. 
				bitShiftAmount++;
				if ((((arrayIndex + theItem.bitOffset + 1) % 8) === 0) 
                    || (arrayIndex == theItem.arrayLength - 1)) {
					thePointer += theItem.dtypelen;
					bitShiftAmount = 0;
				}
			} else {
				// Щоразу додавайте до покажчика.
				thePointer += theItem.dtypelen;
			}
		}
	} else {
		// Єдине значення.
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
					theItem.value = (((theItem.byteBuffer.readUInt8(thePointer) 
                                       >> (theItem.bitOffset)) & 1) ? true : false);
					break;
				case "B":
				case "BYTE":
					// Наразі немає підтримки для знакових 8 біт. 
                    // Це не так часто зустрічається в Siemens. 
					theItem.value = theItem.byteBuffer.readUInt8(thePointer);
					break;
				case "S":
				case "STRING":
					strlen = theItem.byteBuffer.readUInt8(thePointer+1);
					theItem.value = '';
					for (var charOffset = 2; charOffset < theItem.dtypelen 
                         && (charOffset - 2) < strlen; charOffset++) {
						// скажімо, strlen = 1 (рядок з одного символу), 
                        // цей символ має arrayIndex 2. 
						// Convert to string.
						theItem.value += String.fromCharCode(
                            theItem.byteBuffer.readUInt8(thePointer+charOffset));
					}
					break;
				case "C":
				case "CHAR":
					// Наразі немає підтримки для знакових 8 біт. 
                    // Це не так часто зустрічається в Siemens. 
					theItem.value = String.fromCharCode(
                        theItem.byteBuffer.readUInt8(thePointer));
					break;
				case "TIMER":
				case "COUNTER":
					theItem.value = theItem.byteBuffer.readInt16BE(thePointer);
					break;
				default:
					outputLog("Невідомий тип даних у відповіді - ніколи не" + 
                              " повинно статися. Треба було зловити раніше.   " 
                              + theItem.datatype);
					return 0;
			}
		}
		thePointer += theItem.dtypelen;
	}
	// Непарне число. З протоколом S7 ми запитуємо лише парну кількість байтів. 
    // Отже, буде байт-заповнювач.
	if (((thePointer) % 2)) { 
		thePointer += 1;
	}

	return thePointer; // Should maybe return a value now???
}
```

[<-- До опису бібліотеки](README.md) 





