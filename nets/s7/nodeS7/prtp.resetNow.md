[<-- До опису бібліотеки](README.md) 

## resetNow

```js
NodeS7.prototype.resetNow = function() {
	var self = this;
	self.isoConnectionState = 0;
	self.isoclient.end();
	outputLog('ResetNOW is happening', 0, self.connectionID);
	self.resetPending = false;
	// In some cases, we can have a timeout scheduled for a reset, 
    // but we don't want to call it again in that case.
	// We only want to call a reset just as we are returning values.  
    // Otherwise, we will get asked to read 
    // more values and we will "break our promise" to always return something when asked.
	if (typeof (self.resetTimeout) !== 'undefined') {
		clearTimeout(self.resetTimeout);
		self.resetTimeout = undefined;
		outputLog('Clearing an earlier scheduled reset', 0, self.connectionID);
	}
}
```





[<-- До опису бібліотеки](README.md) 





