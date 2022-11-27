[<-- До опису бібліотеки](README.md) 

## findWriteIndexOfSeqNum

Шукає `seqNum` у властивості елементу в масиві пакетів `writePacketArray` . 

```js
NodeS7.prototype.findWriteIndexOfSeqNum = function(seqNum) {
	var self = this, packetCounter;
	for (packetCounter = 0; packetCounter < self.writePacketArray.length; packetCounter++) {
		if (self.writePacketArray[packetCounter].seqNum == seqNum) {
			return packetCounter;
		}
	}
	return undefined;
}
```



[<-- До опису бібліотеки](README.md) 





