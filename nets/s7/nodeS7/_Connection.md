# Підключення по ISO on TCP

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

## connectError

Обробник підключення по TCP

```js
NodeS7.prototype.connectError = function(e) {
	var self = this;
	// Note that a TCP connection timeout error will appear here.  
    // An ISO connection timeout error is a packet timeout.
	outputLog('We Caught a connect error ' + e.code, 0, self.connectionID);
	if ((!self.connectCBIssued) && (typeof (self.connectCallback) === "function")) {
		self.connectCBIssued = true;
		self.connectCallback(e);
	}
	self.isoConnectionState = 0;
}
```

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

## onClientDisconnect

```js
NodeS7.prototype.onClientDisconnect = function() {
	var self = this;
	outputLog('ISO-on-TCP connection DISCONNECTED.', 0, self.connectionID);

	// We issue the callback here for Trela/Honcho - in some cases TCP connects, 
    // and ISO-on-TCP doesn't.
	// If this is the case we need to issue the Connect CB in order to keep trying.
	if ((!self.connectCBIssued) && (typeof (self.connectCallback) === "function")) {
		self.connectCBIssued = true;
		self.connectCallback("Error - TCP connected, ISO didn't");
	}

	// We used to call self.connectionCleanup() - in other words we would give up.
	// However - realize that this event is called when the OTHER END 
    // of the connection sends a FIN packet.
	// Certain situations (download user program to mem card on S7-400, 
    // pop memory card out of S7-300, both with NetLink) cause this to happen.
	// So now, let's try a "connetionReset".  This way, we are guaranteed to return
    // values (or bad) and reset at the proper time.
	// self.connectionCleanup();
	self.connectionReset();
}
```

## onClientClose

```js
NodeS7.prototype.onClientClose = function() {
	var self = this;
    // clean up the connection now the socket has closed
		// We used to call self.connectionCleanup() here, but it caused problems.
		// However - realize that this event is also called when the OTHER END of the connection sends a FIN packet.
		// Certain situations (download user program to mem card on S7-400, pop memory card out of S7-300, both with NetLink) cause this to happen.
		// So now, let's try a "connetionReset".  This way, we are guaranteed to return values (even if bad) and reset at the proper time.
		// Without this, client applications had to be prepared for a read/write not returning.
	self.connectionReset();

    // initiate the callback stored by dropConnection
    if (self.dropConnectionCallback) {
        self.dropConnectionCallback();
        // prevent any possiblity of the callback being called twice
        self.dropConnectionCallback = null;
        // and cancel the timeout
        clearTimeout(self.dropConnectionTimer);
    }
}
```

## connectionReset

```js
NodeS7.prototype.connectionReset = function() {
	var self = this;
	self.isoConnectionState = 0;
	self.resetPending = true;
	outputLog('ConnectionReset has been called to set the reset as pending', 0,
              self.connectionID);
	if (!self.isReading() 
        && !self.isWriting() 
        && !self.writeInQueue 
        && typeof(self.resetTimeout) === 'undefined') 
    	{ // We can no longer logically ignore writes here
			self.resetTimeout = setTimeout(function() {
				outputLog('Timed reset has happened. Ideally this would" +
                          " never be called as reset should be completed" + 
                          "when done r/w.',0,self.connectionID);
			self.resetNow.apply(self, arguments);
		}, 3500);  // Increased to 3500 to prevent problems with packet timeouts
	}
	// We wait until read() is called again to re-connect.
}
```

## resetNow

```js
NodeS7.prototype.resetNow = function() {
	var self = this;
	self.isoConnectionState = 0;
	self.isoclient.end();
	outputLog('ResetNOW is happening', 0, self.connectionID);
	self.resetPending = false;
	// In some cases, we can have a timeout scheduled for a reset, 
    // but we don't want to call it again in that case.
	// We only want to call a reset just as we are returning values.  
    // Otherwise, we will get asked to read 
    // more values and we will "break our promise" to always return something when asked.
	if (typeof (self.resetTimeout) !== 'undefined') {
		clearTimeout(self.resetTimeout);
		self.resetTimeout = undefined;
		outputLog('Clearing an earlier scheduled reset', 0, self.connectionID);
	}
}
```

## connectionCleanup

```js
NodeS7.prototype.connectionCleanup = function() {
	var self = this;
	self.isoConnectionState = 0;
	outputLog('Connection cleanup is happening', 0, self.connectionID);
	if (typeof (self.isoclient) !== "undefined") {
		// destroy the socket connection
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

## onISOConnectReply

```js
NodeS7.prototype.onISOConnectReply = function(data) {
	var self = this;
	self.isoclient.removeAllListeners('data'); //self.onISOConnectReply);
	//self.isoclient.removeAllListeners('error'); Avoid removing the calback before setting it again

	clearTimeout(self.connectTimeout);

	// ignore if we're not expecting it - prevents write after end exception as of #80
	if (self.isoConnectionState != 2) { 
		outputLog('Ignoring ISO connect reply, expecting isoConnectionState of 2, is currently ' + self.isoConnectionState, 0, self.connectionID);
		return; 
	}

	// Track the connection state
	self.isoConnectionState = 3;  // 3 = ISO-ON-TCP connected, Wait for PDU response.

	// Expected length is from packet sniffing - some applications may be different, especially using routing - not considered yet.
	if (data.readInt16BE(2) !== data.length || data.length < 22 || data[5] !== 0xd0 || data[4] !== (data.length - 5)) {
		outputLog('INVALID PACKET or CONNECTION REFUSED - DISCONNECTING');
		outputLog(data);
		outputLog('TPKT Length From Header is ' + data.readInt16BE(2) + ' and RCV buffer length is ' + data.length + ' and COTP length is ' + data.readUInt8(4) + ' and data[5] is ' + data[5]);
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

