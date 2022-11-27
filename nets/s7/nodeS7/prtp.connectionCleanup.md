[<-- До опису бібліотеки](README.md) 

# connectionCleanup

Очищає попереднє з'єднання сокету

```js
NodeS7.prototype.connectionCleanup = function() {
	var self = this;
	self.isoConnectionState = 0;
	outputLog('Connection cleanup is happening', 0, self.connectionID);
	if (typeof (self.isoclient) !== "undefined") {
		// очисити з'єднання сокету
		self.isoclient.destroy();
		self.isoclient.removeAllListeners('data');
		self.isoclient.removeAllListeners('error');
		self.isoclient.removeAllListeners('connect');
		self.isoclient.removeAllListeners('end');
		self.isoclient.removeAllListeners('close');
		self.isoclient.on('error',function() {
			outputLog('TCP socket error following connection cleanup');
		});
	}
	clearTimeout(self.connectTimeout);
	clearTimeout(self.PDUTimeout);
	self.clearReadPacketTimeouts();  // Note this clears timeouts.
	self.clearWritePacketTimeouts();  // Note this clears timeouts.
}
```





[<-- До опису бібліотеки](README.md) 





