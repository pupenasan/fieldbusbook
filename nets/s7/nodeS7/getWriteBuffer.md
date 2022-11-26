[<-- До опису бібліотеки](README.md) 

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





[<-- До опису бібліотеки](README.md) 





