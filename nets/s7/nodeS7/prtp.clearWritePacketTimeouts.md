[<-- До опису бібліотеки](README.md) 

## clearWritePacketTimeouts

```js
NodeS7.prototype.clearWritePacketTimeouts = function() {
	var self = this, i;
	outputLog('Clearing write PacketTimeouts', 1, self.connectionID);
	// Before we initialize the self.readPacketArray, we need to loop through all of them and clear timeouts.
	for (i = 0; i < self.writePacketArray.length; i++) {
		clearTimeout(self.writePacketArray[i].timeout);
		self.writePacketArray[i].sent = false;
		self.writePacketArray[i].rcvd = false;
	}
}
```





[<-- До опису бібліотеки](README.md) 





