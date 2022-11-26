[<-- До опису бібліотеки](README.md) 

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
```





[<-- До опису бібліотеки](README.md) 





