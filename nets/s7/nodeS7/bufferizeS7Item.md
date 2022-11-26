[<-- До опису бібліотеки](README.md) 

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





[<-- До опису бібліотеки](README.md) 





