[<-- До опису бібліотеки](README.md) 

## initiateConnection

підключеється до ПЛК з адресою та параметрами

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





