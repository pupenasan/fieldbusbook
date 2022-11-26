[<-- До опису бібліотеки](README.md) 

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





[<-- До опису бібліотеки](README.md) 





