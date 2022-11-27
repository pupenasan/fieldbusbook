[<-- До опису бібліотеки](README.md) 

# onPDUReply



```js
NodeS7.prototype.onPDUReply = function(theData) {
	var self = this;
	self.isoclient.removeAllListeners('data');
	self.isoclient.removeAllListeners('error');

	clearTimeout(self.PDUTimeout);

	var data=checkRFCData(theData);

	if(data==="fastACK"){
		// Прочитайте ще раз і дочекайтеся запитуваних даних
		outputLog('Fast Acknowledge received.', 0, self.connectionID);
		self.isoclient.removeAllListeners('error');
		self.isoclient.removeAllListeners('data');
		self.isoclient.on('data', function() {
			self.onPDUReply.apply(self, arguments);
		});
		self.isoclient.on('error', function() {
			self.readWriteError.apply(self, arguments);
		});
	} else if ((data[4] + 1 + 12 + data.readInt16BE(13) === data.readInt16BE(2) - 4)){
        // дійсна довжина пакета FA (Fast Acknodledge) +S7: 
        // ISO_Length + ISO_LengthItself + S7Com_Header + S7Com_Header_ParameterLength 
        // === TPKT_Length - 4
		// Все добре... продовжуй
		// Відстежуйте стан підключення
		self.isoConnectionState = 4;  // 4 = Received PDU response, good to go
		// Нам потрібно обнулити це тут, 
        // оскільки воно може стати від’ємним, коли не підключено
        self.parallelJobsNow = 0;     

		var partnerMaxParallel1 = data.readInt16BE(21);
		var partnerMaxParallel2 = data.readInt16BE(23);
		var partnerPDU = data.readInt16BE(25);

		self.maxParallel = self.requestMaxParallel;

		if (partnerMaxParallel1 < self.requestMaxParallel) {
			self.maxParallel = partnerMaxParallel1;
		}
		if (partnerMaxParallel2 < self.requestMaxParallel) {
			self.maxParallel = partnerMaxParallel2;
		}
		if (partnerPDU < self.requestMaxPDU) {
			self.maxPDU = partnerPDU;
		} else {
			self.maxPDU = self.requestMaxPDU;
		}

		outputLog('Received PDU Response - Proceeding with PDU ' 
                  + self.maxPDU + ' and ' 
                  + self.maxParallel + ' max parallel connections.', 
                  0, self.connectionID);
		// Нам потрібно переконатися, що ми не додаємо цю подію щоразу, 
        // якщо ми викликаємо її в даних.
        self.isoclient.on('data', function() {
			self.onResponse.apply(self, arguments);
		});
        // Можливо, потрібно видалити прослуховувач self.connecterror
		self.isoclient.on('error', function() {
			self.readWriteError.apply(self, arguments);
		}); 
		if ((!self.connectCBIssued) && (typeof (self.connectCallback) === "function")) {
			self.connectCBIssued = true;
			self.connectCallback();
		}
	} else {
		outputLog('INVALID Telegram ', 0, self.connectionID);
		outputLog('Byte 0 From Header is ' + theData[0] + ' it has to be 0x03, Byte 5 From Header is  ' + theData[5] + ' and it has to be 0x0F ', 0, self.connectionID);
		outputLog('INVALID PDU RESPONSE or CONNECTION REFUSED - DISCONNECTING', 0, self.connectionID);
		outputLog('TPKT Length From Header is ' + theData.readInt16BE(2) + ' and RCV buffer length is ' + theData.length + ' and COTP length is ' + theData.readUInt8(4) + ' and data[6] is ' + theData[6], 0, self.connectionID);
		outputLog(theData);
		self.isoclient.end();
		clearTimeout(self.reconnectTimer);
		self.reconnectTimer = setTimeout(function() {
			self.connectNow.apply(self, arguments);
		}, 2000, self.connectionParams);
		return null;
	}
}
```





[<-- До опису бібліотеки](README.md) 





