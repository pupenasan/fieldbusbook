[<-- До опису бібліотеки](README.md) 

# findReadIndexOfSeqNum

Шукає `seqNum` у властивості елементу в масиві пакетів `readPacketArray` . 

```js
NodeS7.prototype.findReadIndexOfSeqNum = function(seqNum) {
	var self = this, packetCounter;
	for (packetCounter = 0; packetCounter < self.readPacketArray.length; packetCounter++) {
		if (self.readPacketArray[packetCounter].seqNum == seqNum) {
			return packetCounter;
		}
	}
	return undefined;
}
```

[<-- До опису бібліотеки](README.md) 





