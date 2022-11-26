[<-- До опису бібліотеки](README.md) 

## dropConnection

```js
NodeS7.prototype.dropConnection = function(callback) {
	var self = this;

	// prevents triggering reconnection even after calling dropConnection (fixes #70)
	clearTimeout(self.reconnectTimer);
	clearTimeout(self.rereadTimer);
	clearTimeout(self.connectTimeout);
	clearTimeout(self.PDUTimeout);
	self.reconnectTimer = undefined;
	self.rereadTimer = undefined;
	self.connectTimeout = undefined;
	self.PDUTimeout = undefined;

	if (typeof (self.isoclient) !== 'undefined') {
		// store the callback and request and end to the connection
		self.dropConnectionCallback = callback;
		self.isoclient.end();
		// now wait for 'on close' event to trigger connection cleanup
		// but also start a timer to destroy the connection in case 
        // we do not receive the close
		self.dropConnectionTimer = setTimeout(function() {
			if (self.dropConnectionCallback) {
				// clean up the connection now the socket has closed
				self.connectionCleanup();
				// initate the callback
				self.dropConnectionCallback();
				// prevent any possiblity of the callback being called twice
				self.dropConnectionCallback = null;
			}
		}, 2500);
	} else {
		// if client not active, then callback immediately
		callback();
	}
}
```





[<-- До опису бібліотеки](README.md) 





