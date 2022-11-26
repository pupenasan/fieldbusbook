[<-- До опису бібліотеки](README.md) 

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





[<-- До опису бібліотеки](README.md) 





