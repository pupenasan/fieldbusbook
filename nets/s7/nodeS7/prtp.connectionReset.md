[<-- До опису бібліотеки](README.md) 

## connectionReset

```js
NodeS7.prototype.connectionReset = function() {
	var self = this;
	self.isoConnectionState = 0;
	self.resetPending = true;
	outputLog('ConnectionReset has been called to set the reset as pending', 0,
              self.connectionID);
	if (!self.isReading() 
        && !self.isWriting() 
        && !self.writeInQueue 
        && typeof(self.resetTimeout) === 'undefined') 
    	{ // We can no longer logically ignore writes here
			self.resetTimeout = setTimeout(function() {
				outputLog('Timed reset has happened. Ideally this would" +
                          " never be called as reset should be completed" + 
                          "when done r/w.',0,self.connectionID);
			self.resetNow.apply(self, arguments);
		}, 3500);  // Increased to 3500 to prevent problems with packet timeouts
	}
	// We wait until read() is called again to re-connect.
}
```





[<-- До опису бібліотеки](README.md) 





