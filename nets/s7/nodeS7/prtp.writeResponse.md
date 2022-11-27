[<-- До опису бібліотеки](README.md) 

# writeResponse

```
writeResponse (data, foundSeqNum)
```



```js
NodeS7.prototype.writeResponse = function(data, foundSeqNum) {
	var self = this, dataPointer = 21, i, anyBadQualities;

	for (var itemCount = 0; 
         itemCount < self.writePacketArray[foundSeqNum].itemList.length; itemCount++) {
		dataPointer = processS7WriteItem(data, 
		self.writePacketArray[foundSeqNum].itemList[itemCount],
        dataPointer);
		if (!dataPointer) {
			outputLog('Зупинка обробки пакета відповіді на запис" + 
                      "через невиправну помилку пакета');
			break;
		}
	}

	// Make a note of the time it took the PLC to process the request.
	self.writePacketArray[foundSeqNum].reqTime = process.hrtime(self.writePacketArray[foundSeqNum].reqTime);
	outputLog('Time is ' + self.writePacketArray[foundSeqNum].reqTime[0] + ' seconds and ' + Math.round(self.writePacketArray[foundSeqNum].reqTime[1] * 10 / 1e6) / 10 + ' ms.', 1, self.connectionID);

	//	self.writePacketArray.splice(foundSeqNum, 1);
	if (!self.writePacketArray[foundSeqNum].rcvd) {
		self.writePacketArray[foundSeqNum].rcvd = true;
		self.parallelJobsNow--;
	}
	clearTimeout(self.writePacketArray[foundSeqNum].timeout);

	if (!self.writePacketArray.every(doneSending)) {
		outputLog("Not done sending - sending more packets from writeResponse",1,self.connectionID);
		self.sendWritePacket();
	} else {
		outputLog("Received all packets in writeResponse",1,self.connectionID);
		for (i = 0; i < self.writePacketArray.length; i++) {
			self.writePacketArray[i].sent = false;
			self.writePacketArray[i].rcvd = false;
		}

		anyBadQualities = false;

		for (i = 0; i < self.globalWriteBlockList.length; i++) {
			// Post-process the write code and apply the quality.
			// Loop through the global block list...
			writePostProcess(self.globalWriteBlockList[i]);
			for (var k = 0; k < self.globalWriteBlockList[i].itemReference.length; k++) {
				outputLog(self.globalWriteBlockList[i].itemReference[k].addr + ' write completed with quality ' + self.globalWriteBlockList[i].itemReference[k].writeQuality, 0, self.connectionID);
				if (!isQualityOK(self.globalWriteBlockList[i].itemReference[k].writeQuality)) {
					anyBadQualities = true;
				}
			}
//			outputLog(self.globalWriteBlockList[i].addr + ' write completed with quality ' + self.globalWriteBlockList[i].writeQuality, 1, self.connectionID);
			if (!isQualityOK(self.globalWriteBlockList[i].writeQuality)) { anyBadQualities = true; }
		}
		if (self.resetPending) {
			outputLog('Calling reset from writeResponse as there is one pending',0,self.connectionID);
			self.resetNow();
		}
		if (self.isoConnectionState === 0) {
			self.connectNow(self.connectionParams, false);
		}
		outputLog('We are calling back our writeDoneCallback.',1,self.connectionID);
		if (typeof(self.writeDoneCallback) === 'function') {
			self.writeDoneCallback(anyBadQualities);
		}
	}
}
```





[<-- До опису бібліотеки](README.md) 

