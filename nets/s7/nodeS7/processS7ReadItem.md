[<-- До опису бібліотеки](README.md) 

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





[<-- До опису бібліотеки](README.md) 





