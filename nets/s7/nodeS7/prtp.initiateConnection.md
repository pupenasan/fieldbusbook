[<-- До опису бібліотеки](README.md) 

# initiateConnection

## Опис

`initiateConnection (cParam, callback)` - підключеється до ПЛК з адресою та параметрами:

- `cParam` - параметри підключення
- `callback` - функція, що викликається 

```js
cParam : { port: 102, 
  host: '192.168.8.106', 
  rack: 0,//для remoteTSAP
  slot: 2,//для remoteTSAP     
  localTSAP = 0x0100,//альтернативний варіант вказівки безпосередньо
  remoteTSAP = 0x0200,//альтернативний варіант вказівки безпосередньо
  connection_name  = cParam.host + " S" + self.slot //connectionID
  timeout: 8000,// час очікування TCP в мс
  doNotOptimize: true,//
}     
```

Приклад використання:

```js
conn.initiateConnection(
    { port: 102, host: '192.168.56.102', rack: 0, slot: 2, debug: true }, 
    connected); // slot 2 for 300/400, slot 1 for 1200/1500, change debug to true to get more info
```

## Реалізація

```js
NodeS7.prototype.initiateConnection = function(cParam, callback) {
	var self = this;
	if (cParam === undefined) { cParam = { port: 102, host: '192.168.8.106' }; }
	outputLog('Initiate Called - Connecting to PLC with address and parameters:');
	outputLog(cParam);
	if (typeof (cParam.rack) !== 'undefined') {
		self.rack = cParam.rack;
	}
	if (typeof (cParam.slot) !== 'undefined') {
		self.slot = cParam.slot;
	}
	if (typeof (cParam.localTSAP) !== 'undefined') {
		self.localTSAP = cParam.localTSAP;
	}
	if (typeof (cParam.remoteTSAP) !== 'undefined') {
		self.remoteTSAP = cParam.remoteTSAP;
	}
	if (typeof (cParam.connection_name) === 'undefined') {
		self.connectionID = cParam.host + " S" + self.slot;
	} else {
		self.connectionID = cParam.connection_name;
	}
	if (typeof (cParam.doNotOptimize) !== 'undefined') {
		self.doNotOptimize = cParam.doNotOptimize;
	}
	if (typeof (cParam.timeout) !== 'undefined') { // Added in 0.3.17 to ensure packets don't timeout at 1500ms if the user has specified a timeout externally.
		self.globalTimeout = cParam.timeout;
	}
	self.connectionParams = cParam;
	self.connectCallback = callback;
	self.connectCBIssued = false;
	self.connectNow(self.connectionParams, false);
}
```





[<-- До опису бібліотеки](README.md) 





