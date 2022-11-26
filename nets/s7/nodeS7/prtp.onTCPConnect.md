[<-- До опису бібліотеки](README.md) 

## onTCPConnect

```js
NodeS7.prototype.onTCPConnect = function() {
	var self = this, connBuf;
	outputLog('TCP Connection Established to ' + self.isoclient.remoteAddress + ' on port ' + self.isoclient.remotePort, 0, self.connectionID);
	outputLog('Will attempt ISO-on-TCP connection', 0, self.connectionID);

	// Відстежує стан підключення
	self.isoConnectionState = 2;  // 2 = TCP connected, wait for ISO connection confirmation
	// Send an ISO-on-TCP connection request.
	self.connectTimeout = setTimeout(function() {
		self.packetTimeout.apply(self, arguments);
	}, self.globalTimeout, "connect");

	connBuf = self.connectReq.slice();

	if(self.localTSAP !== null && self.remoteTSAP !== null) {
		outputLog('Using localTSAP [0x' + self.localTSAP.toString(16) + '] and remoteTSAP [0x' + self.remoteTSAP.toString(16) + ']', 0, self.connectionID);
		connBuf.writeUInt16BE(self.localTSAP, 16)
		connBuf.writeUInt16BE(self.remoteTSAP, 20)
	} else {
		outputLog('Using rack [' + self.rack + '] and slot [' + self.slot + ']', 0, self.connectionID);
		connBuf[21] = self.rack * 32 + self.slot;
	}

	self.isoclient.write(connBuf);

	// Listen for a reply.
	self.isoclient.on('data', function() {
		self.onISOConnectReply.apply(self, arguments);
	});

	// Hook up the event that fires on disconnect
	self.isoclient.on('end', function() {
		self.onClientDisconnect.apply(self, arguments);
	});

    // listen for close (caused by us sending an end)
	self.isoclient.on('close', function() {
		self.onClientClose.apply(self, arguments);

	});
}
```





[<-- До опису бібліотеки](README.md) 





