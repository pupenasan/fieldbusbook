[<-- До опису бібліотеки](README.md) 

# onISOConnectReply

Очікує відповіді від запиту на підключення по ISO/TCP, при позитивній відповіді запускає запит на переговори `negotiatePDU` та при позитивній відповіді викликає [onPDUReply](prtp.onPDUReply.md)

```js
NodeS7.prototype.onISOConnectReply = function(data) {
	var self = this;
	self.isoclient.removeAllListeners('data'); 
	clearTimeout(self.connectTimeout);

	// ігнорувати, якщо ми цього не очікуємо - запобігає винятку запису 
    // після завершення з #80
	if (self.isoConnectionState != 2) { 
		outputLog('Ignoring ISO connect reply, expecting isoConnectionState of 2," + 
                  " is currently ' + self.isoConnectionState, 0, self.connectionID);
		return; 
	}

	// Відстежуйте стан підключення
	self.isoConnectionState = 3;  // 3 = ISO-ON-TCP connected, Wait for PDU response.

   // Очікувана довжина залежить від аналізу пакетів 
    // - деякі програми можуть відрізнятися, особливо з використанням маршрутизації 
    // - ще не розглядається.
	if (data.readInt16BE(2) !== data.length 
        || data.length < 22 
        || data[5] !== 0xd0 
        || data[4] !== (data.length - 5)) {
		outputLog('INVALID PACKET or CONNECTION REFUSED - DISCONNECTING');
		outputLog(data);
		outputLog('TPKT Length From Header is ' + data.readInt16BE(2) 
                  + ' and RCV buffer length is ' + data.length 
                  + ' and COTP length is ' + data.readUInt8(4) 
                  + ' and data[5] is ' + data[5]);
		self.connectionReset();
		return null;
	}

	outputLog('ISO-on-TCP Connection Confirm Packet Received', 0, self.connectionID);

	self.negotiatePDU.writeInt16BE(self.requestMaxParallel, 19);
	self.negotiatePDU.writeInt16BE(self.requestMaxParallel, 21);
	self.negotiatePDU.writeInt16BE(self.requestMaxPDU, 23);

	self.PDUTimeout = setTimeout(function() {
		self.packetTimeout.apply(self, arguments);
	}, self.globalTimeout, "PDU");

	self.isoclient.write(self.negotiatePDU.slice(0, 25));
	self.isoclient.on('data', function() {
		self.onPDUReply.apply(self, arguments);
	});
    self.isoclient.removeAllListeners('error');
	self.isoclient.on('error', function() {
		self.readWriteError.apply(self, arguments);
	});
}
```





[<-- До опису бібліотеки](README.md) 





