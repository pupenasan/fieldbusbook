[<-- До опису бібліотеки](README.md) 

## clearReadPacketTimeouts

```js
NodeS7.prototype.clearReadPacketTimeouts = function() {
	var self = this, i;
	outputLog('Clearing read PacketTimeouts', 1, self.connectionID);
	// Before we initialize the self.readPacketArray, we need to loop through all of them and clear timeouts.
	for (i = 0; i < self.readPacketArray.length; i++) {
		clearTimeout(self.readPacketArray[i].timeout);
		self.readPacketArray[i].sent = false;
		self.readPacketArray[i].rcvd = false;
	}
}
```





[<-- До опису бібліотеки](README.md) 





