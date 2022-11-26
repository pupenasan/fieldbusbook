[<-- До опису бібліотеки](README.md) 

## packetTimeout

```js
NodeS7.prototype.packetTimeout = function(packetType, packetSeqNum) {
	var self = this;
	outputLog('PacketTimeout called with type ' + packetType + 
              ' and seq ' + packetSeqNum, 1, self.connectionID);
	if (packetType === "connect") {
		outputLog("TIMED OUT connecting to the PLC - Disconnecting", 
                  0, self.connectionID);
		outputLog("Wait for 2 seconds then try again.", 0, self.connectionID);
		self.connectionReset();
		outputLog("Scheduling a reconnect from packetTimeout, connect type", 
                  0, self.connectionID);
		clearTimeout(self.reconnectTimer);
		self.reconnectTimer = setTimeout(function() {
			outputLog("The scheduled reconnect from packetTimeout, connect type," + 
                      "is happening now", 0, self.connectionID);
			if (self.isoConnectionState === 0) {
				self.connectNow.apply(self, arguments);
			}
		}, 2000, self.connectionParams);
		return undefined;
	}
	if (packetType === "PDU") {
		outputLog("TIMED OUT waiting for PDU reply packet from PLC - Disconnecting");
		outputLog("Wait for 2 seconds then try again.", 0, self.connectionID);
		self.connectionReset();
		outputLog("Scheduling a reconnect from packetTimeout, connect type", 
                  0, self.connectionID);
		clearTimeout(self.reconnectTimer);
		self.reconnectTimer = setTimeout(function() {
			outputLog("The scheduled reconnect from packetTimeout," +
                      " PDU type, is happening now", 0, self.connectionID);
			self.connectNow.apply(self, arguments);
		}, 2000, self.connectionParams);
		return undefined;
	}
	if (packetType === "read") {
		outputLog("READ TIMEOUT on sequence number " + packetSeqNum, 
                  0, self.connectionID);
		if (self.isoConnectionState === 4) { 
            // Reset before calling writeResponse so ResetNow will take place this cycle 
			outputLog("ConnectionReset from read packet timeout.", 0, self.connectionID);
			self.connectionReset();
		}
		self.readResponse(undefined, self.findReadIndexOfSeqNum(packetSeqNum));
		return undefined;
	}
	if (packetType === "write") {
		outputLog("WRITE TIMEOUT on sequence number " + packetSeqNum, 
                  0, self.connectionID);
		if (self.isoConnectionState === 4) { 
            // Reset before calling writeResponse so ResetNow will take place this cycle 
			outputLog("ConnectionReset from write packet timeout.", 0,
                      self.connectionID);
			self.connectionReset();
		}
		self.writeResponse(undefined, self.findWriteIndexOfSeqNum(packetSeqNum));
		return undefined;
	}
	outputLog("Unknown timeout error.  Nothing was done - this shouldn't happen.");
}
```





[<-- До опису бібліотеки](README.md) 





