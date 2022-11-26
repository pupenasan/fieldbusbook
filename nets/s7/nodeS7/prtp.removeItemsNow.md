[<-- До опису бібліотеки](README.md) 

```js
NodeS7.prototype.removeItemsNow = function(arg) {
	var self = this, i;
	if (typeof arg === "undefined") {
		self.polledReadBlockList = [];
	} else if (typeof arg === "string") {
		for (i = 0; i < self.polledReadBlockList.length; i++) {
			outputLog('TCBA ' + self.translationCB(arg));
			if (self.polledReadBlockList[i].addr === self.translationCB(arg)) {
				outputLog('Splicing');
				self.polledReadBlockList.splice(i, 1);
			}
		}
	} else if (Array.isArray(arg)) {
		for (i = 0; i < self.polledReadBlockList.length; i++) {
			for (var j = 0; j < arg.length; j++) {
				if (self.polledReadBlockList[i].addr === self.translationCB(arg[j])) {
					self.polledReadBlockList.splice(i, 1);
				}
			}
		}
	}
	self.readPacketValid = false;
	//	self.prepareReadPacket();
}
```





[<-- До опису бібліотеки](README.md) 





