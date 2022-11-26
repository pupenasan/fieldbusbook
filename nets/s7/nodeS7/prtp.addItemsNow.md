[<-- До опису бібліотеки](README.md) 

```js
NodeS7.prototype.addItemsNow = function(arg) {
	var self = this, i;
	outputLog("Adding " + arg, 0, self.connectionID);
	if (typeof (arg) === "string" && arg !== "_COMMERR") {
		self.polledReadBlockList.push(stringToS7Addr(
            self.translationCB(arg), arg, self.connectionParams));
	} else if (Array.isArray(arg)) {
		for (i = 0; i < arg.length; i++) {
			if (typeof (arg[i]) === "string" && arg[i] !== "_COMMERR") {
				self.polledReadBlockList.push(
                    stringToS7Addr(self.translationCB(arg[i]), 
                                   arg[i], self.connectionParams));
			}
		}
	}

	// Validity check.
	for (i = self.polledReadBlockList.length - 1; i >= 0; i--) {
		if (self.polledReadBlockList[i] === undefined) {
			self.polledReadBlockList.splice(i, 1);
			outputLog("Dropping an undefined request item.", 0, self.connectionID);
		}
	}
	//	self.prepareReadPacket();
	self.readPacketValid = false;
}
```





[<-- До опису бібліотеки](README.md) 





