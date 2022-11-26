[<-- До опису бібліотеки](README.md) 

## getNextSeqNum

```js
NodeS7.prototype.getNextSeqNum = function() {
	var self = this;

	self.masterSequenceNumber += 1;
	if (self.masterSequenceNumber > 32767) {
		self.masterSequenceNumber = 1;
	}
	outputLog('seqNum is ' + self.masterSequenceNumber, 1, self.connectionID);
	return self.masterSequenceNumber;
}
```





[<-- До опису бібліотеки](README.md) 





