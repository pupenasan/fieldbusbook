[<-- До опису бібліотеки](README.md) 

## connectError

Обробник підключення по TCP

```js
NodeS7.prototype.connectError = function(e) {
	var self = this;
	// Note that a TCP connection timeout error will appear here.  
    // An ISO connection timeout error is a packet timeout.
	outputLog('We Caught a connect error ' + e.code, 0, self.connectionID);
	if ((!self.connectCBIssued) && (typeof (self.connectCallback) === "function")) {
		self.connectCBIssued = true;
		self.connectCallback(e);
	}
	self.isoConnectionState = 0;
}
```





[<-- До опису бібліотеки](README.md) 





