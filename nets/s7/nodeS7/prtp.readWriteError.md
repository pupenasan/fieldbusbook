[<-- До опису бібліотеки](README.md) 

## readWriteError

```js
NodeS7.prototype.readWriteError = function(e) {
	var self = this;
	outputLog('We Caught a read/write error ' + e.code + ' - will DISCONNECT and attempt to reconnect.');
	self.isoConnectionState = 0;
	self.connectionReset();
}
```





[<-- До опису бібліотеки](README.md) 





