[<-- До опису бібліотеки](README.md) 

# onTCPConnect

Викликається при вдалому підключенні по TCP, для реалізації підключення по ISO/TCP

```js
NodeS7.prototype.onTCPConnect = function() {
	var self = this, connBuf;
	outputLog('TCP Connection Established to ' + 
              self.isoclient.remoteAddress + ' on port ' 
              + self.isoclient.remotePort, 0, self.connectionID);
	outputLog('Will attempt ISO-on-TCP connection', 0, self.connectionID);

	// Відстежує стан підключення
    // 2 = TCP підключено, дочекайтеся підтвердження підключення ISO
	self.isoConnectionState = 2;  
	// Send an ISO-on-TCP connection request.
	self.connectTimeout = setTimeout(function() {
		self.packetTimeout.apply(self, arguments);
	}, self.globalTimeout, "connect");

	connBuf = self.connectReq.slice();

	if(self.localTSAP !== null && self.remoteTSAP !== null) {
		outputLog('Using localTSAP [0x' + self.localTSAP.toString(16) + 
                  '] and remoteTSAP [0x' + self.remoteTSAP.toString(16) + ']',
                  0, self.connectionID);
		connBuf.writeUInt16BE(self.localTSAP, 16)
		connBuf.writeUInt16BE(self.remoteTSAP, 20)
	} else {
		outputLog('Using rack [' + self.rack + '] and slot [' + 
                  self.slot + ']', 0, self.connectionID);
		connBuf[21] = self.rack * 32 + self.slot;
	}

	self.isoclient.write(connBuf);

	// прослушка відповіді.
	self.isoclient.on('data', function() {
		self.onISOConnectReply.apply(self, arguments);
	});

	// Підключіть подію, яка виникає після відключення
	self.isoclient.on('end', function() {
		self.onClientDisconnect.apply(self, arguments);
	});

    // прослушка закриття (спричинено тим, що ми надсилаємо кінець)
	self.isoclient.on('close', function() {
		self.onClientClose.apply(self, arguments);

	});
}
```





[<-- До опису бібліотеки](README.md) 





