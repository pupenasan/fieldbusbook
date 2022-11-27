[<-- До опису бібліотеки](README.md) 

## connectError

Обробник помилки підключення по TCP

```js
NodeS7.prototype.connectError = function(e) {
	var self = this;
	// Зауважте, що тут з’явиться помилка тайм-ауту з’єднання TCP.  
    // Помилка тайм-ауту підключення ISO – це тайм-аут пакета (packet timeout).
	outputLog('We Caught a connect error ' + e.code, 0, self.connectionID);
	if ((!self.connectCBIssued) && (typeof (self.connectCallback) === "function")) {
		self.connectCBIssued = true;
		self.connectCallback(e);
	}
	self.isoConnectionState = 0;
}
```





[<-- До опису бібліотеки](README.md) 





