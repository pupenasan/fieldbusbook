[<-- До опису бібліотеки](README.md) 

## connectNow

включена функція в initiateConnection, реалізує зєднання 

```js
NodeS7.prototype.connectNow = function(cParam) {
	var self = this;

	// запобігає повторному запуску будь-якого таймера повторного підключення 
	clearTimeout(self.reconnectTimer);
	self.reconnectTimer = undefined;

	// Don't re-trigger.
	if (self.isoConnectionState >= 1) { return; }
	self.connectionCleanup();
        self.isoclient = net.connect(cParam);                                             
        self.isoclient.setTimeout(cParam.timeout || 5000, () => {                         
           self.isoclient.destroy();
          //Використовуйте "USERTIMEOUT", щоб показати різницю між цим та тайм-аутом TCP.
           self.connectError.apply(self, [{ code: 'EUSERTIMEOUT' }]);                     
        });                                                                               
        self.isoclient.once('connect', () => {                                           
            self.isoclient.setTimeout(0);                                                 
            self.onTCPConnect.apply(self, arguments);                                     
        });                                                                               
        self.isoConnectionState = 1;  // 1 = trying to connect  
		self.isoclient.on('error', function() {
            self.connectError.apply(self, arguments);
	});

	outputLog('<initiating a new connection ' + Date() + '>', 1, self.connectionID);
	outputLog('Attempting to connect to host...', 0, self.connectionID);
}
```





[<-- До опису бібліотеки](README.md) 





