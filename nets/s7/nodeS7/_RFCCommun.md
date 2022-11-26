# Комунікаційні функції поверх RFC

## getNextSeqNum

```js
NodeS7.prototype.getNextSeqNum = function() {
	var self = this;

	self.masterSequenceNumber += 1;
	if (self.masterSequenceNumber > 32767) {
		self.masterSequenceNumber = 1;
	}
	outputLog('seqNum is ' + self.masterSequenceNumber, 1, self.connectionID);
	return self.masterSequenceNumber;
}
```

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

## onPDUReply

```js
NodeS7.prototype.onPDUReply = function(theData) {
	var self = this;
	self.isoclient.removeAllListeners('data');
	self.isoclient.removeAllListeners('error');

	clearTimeout(self.PDUTimeout);

	var data=checkRFCData(theData);

	if(data==="fastACK"){
		//Read again and wait for the requested data
		outputLog('Fast Acknowledge received.', 0, self.connectionID);
		self.isoclient.removeAllListeners('error');
		self.isoclient.removeAllListeners('data');
		self.isoclient.on('data', function() {
			self.onPDUReply.apply(self, arguments);
		});
		self.isoclient.on('error', function() {
			self.readWriteError.apply(self, arguments);
		});
	}else if((data[4] + 1 + 12 + data.readInt16BE(13) === data.readInt16BE(2) - 4)){//valid the length of FA+S7 package :  ISO_Length+ISO_LengthItself+S7Com_Header+S7Com_Header_ParameterLength===TPKT_Length-4
		//Everything OK...go on
		// Track the connection state
		self.isoConnectionState = 4;  // 4 = Received PDU response, good to go
		self.parallelJobsNow = 0;     // We need to zero this here as it can go negative when not connected

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

		outputLog('Received PDU Response - Proceeding with PDU ' + self.maxPDU + ' and ' + self.maxParallel + ' max parallel connections.', 0, self.connectionID);
		self.isoclient.on('data', function() {
			self.onResponse.apply(self, arguments);
		});  // We need to make sure we don't add this event every time if we call it on data.
		self.isoclient.on('error', function() {
			self.readWriteError.apply(self, arguments);
		});  // Might want to remove the self.connecterror listener
		//self.isoclient.removeAllListeners('error');
		if ((!self.connectCBIssued) && (typeof (self.connectCallback) === "function")) {
			self.connectCBIssued = true;
			self.connectCallback();
		}
	}else{
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

## clearReadPacketTimeouts

```js
NodeS7.prototype.clearReadPacketTimeouts = function() {
	var self = this, i;
	outputLog('Clearing read PacketTimeouts', 1, self.connectionID);
	// Before we initialize the self.readPacketArray, we need to loop through all of them and clear timeouts.
	for (i = 0; i < self.readPacketArray.length; i++) {
		clearTimeout(self.readPacketArray[i].timeout);
		self.readPacketArray[i].sent = false;
		self.readPacketArray[i].rcvd = false;
	}
}
```

## clearWritePacketTimeouts

```js
NodeS7.prototype.clearWritePacketTimeouts = function() {
	var self = this, i;
	outputLog('Clearing write PacketTimeouts', 1, self.connectionID);
	// Before we initialize the self.readPacketArray, we need to loop through all of them and clear timeouts.
	for (i = 0; i < self.writePacketArray.length; i++) {
		clearTimeout(self.writePacketArray[i].timeout);
		self.writePacketArray[i].sent = false;
		self.writePacketArray[i].rcvd = false;
	}
}
```

## prepareWritePacket

```js
NodeS7.prototype.prepareWritePacket = function() {
	var self = this, i;
	var itemList = self.instantWriteBlockList;
    // Список запитів складається зі списку блоків, 
    // розділеного на фрагменти, доступні для читання PDU.
	var requestList = [];			
	var requestNumber = 0;
	// Сортує елементи за допомогою функції сортування за типом і offset.
	itemList.sort(itemListSorter);

	// Just exit if there are no items.
	if (itemList.length === 0) {
		return undefined;
	}

	// Reinitialize the WriteBlockList
	self.globalWriteBlockList = [];

	// At this time we do not do write optimizations.
	// The reason for this is it is would cause numerous issues depending how the code was written in the PLC.
	// If we write M0.1 and M0.2 then to optimize we would have to write MB0, which also writes 0.0, 0.3, 0.4...
	//
	// I suppose when working with integers, if we write MW0 and MW2, we could write these as one block.
	// But if you really, really want the program to do that, write an array yourself.
	self.globalWriteBlockList[0] = itemList[0];
	self.globalWriteBlockList[0].itemReference = [];
	self.globalWriteBlockList[0].itemReference.push(itemList[0]);

	var thisBlock = 0;
	itemList[0].block = thisBlock;
	var maxByteRequest = 4 * Math.floor((self.maxPDU - 18 - 12) / 4);  // Absolutely must not break a real array into two requests.  Maybe we can extend by two bytes when not DINT/REAL/INT.  But modified now for LREAL.
	maxByteRequest = 8 * Math.floor((self.maxPDU - 18 - 12) / 8);
	//	outputLog("Max Write Length is " + maxByteRequest);

	// Just push the items into blocks and figure out the write buffers
	for (i = 0; i < itemList.length; i++) {
		self.globalWriteBlockList[i] = itemList[i]; // Remember - by reference.
		self.globalWriteBlockList[i].isOptimized = false;
		self.globalWriteBlockList[i].itemReference = [];
		self.globalWriteBlockList[i].itemReference.push(itemList[i]);
		bufferizeS7Item(itemList[i]);
	}

	var thisRequest = 0;

	// Split the blocks into requests, if they're too large.
	for (i = 0; i < self.globalWriteBlockList.length; i++) {
		var startByte = self.globalWriteBlockList[i].offset;
		var remainingLength = self.globalWriteBlockList[i].byteLength;
		var lengthOffset = 0;

		// Always create a request for a self.globalReadBlockList.
		requestList[thisRequest] = self.globalWriteBlockList[i].clone();

		// How many parts?
		self.globalWriteBlockList[i].parts = Math.ceil(self.globalWriteBlockList[i].byteLength / maxByteRequest);
		//		outputLog("self.globalWriteBlockList " + i + " parts is " + self.globalWriteBlockList[i].parts + " offset is " + self.globalWriteBlockList[i].offset + " MBR is " + maxByteRequest);

		self.globalWriteBlockList[i].requestReference = [];

		// If we're optimized...
		for (var j = 0; j < self.globalWriteBlockList[i].parts; j++) {
			requestList[thisRequest] = self.globalWriteBlockList[i].clone();
			self.globalWriteBlockList[i].requestReference.push(requestList[thisRequest]);
			requestList[thisRequest].offset = startByte;
			requestList[thisRequest].byteLength = Math.min(maxByteRequest, remainingLength);
			requestList[thisRequest].byteLengthWithFill = requestList[thisRequest].byteLength;
			if (requestList[thisRequest].byteLengthWithFill % 2) { requestList[thisRequest].byteLengthWithFill += 1; }

			// max

			requestList[thisRequest].writeBuffer = self.globalWriteBlockList[i].writeBuffer.slice(lengthOffset, lengthOffset + requestList[thisRequest].byteLengthWithFill);
			requestList[thisRequest].writeQualityBuffer = self.globalWriteBlockList[i].writeQualityBuffer.slice(lengthOffset, lengthOffset + requestList[thisRequest].byteLengthWithFill);
			lengthOffset += self.globalWriteBlockList[i].requestReference[j].byteLength;

			if (self.globalWriteBlockList[i].parts > 1) {
				requestList[thisRequest].datatype = 'BYTE';
				requestList[thisRequest].dtypelen = 1;
				requestList[thisRequest].arrayLength = requestList[thisRequest].byteLength;//self.globalReadBlockList[thisBlock].byteLength;		(This line shouldn't be needed anymore - shouldn't matter)
			}
			remainingLength -= maxByteRequest;
			thisRequest++;
			startByte += maxByteRequest;
		}
	}

	self.clearWritePacketTimeouts();
	self.writePacketArray = [];

	//	outputLog("GWBL is " + self.globalWriteBlockList.length);


	// Before we initialize the self.writePacketArray, we need to loop through all of them and clear timeouts.

	// The packetizer...
	while (requestNumber < requestList.length) {
		// Set up the read packet
		// Yes this is the same master sequence number shared with the read queue

		var numItems = 0;

		// Maybe this shouldn't really be here?
		self.writeReqHeader.copy(self.writeReq, 0);

		// Packet's length
		var packetWriteLength = 10 + 4;  // 10 byte header and 4 byte param header

		self.writePacketArray.push(new S7Packet());
		var thisPacketNumber = self.writePacketArray.length - 1;
		self.writePacketArray[thisPacketNumber].seqNum = self.getNextSeqNum();
		//		outputLog("Write Sequence Number is " + self.writePacketArray[thisPacketNumber].seqNum);

		self.writePacketArray[thisPacketNumber].itemList = [];  // Initialize as array.

		for (i = requestNumber; i < requestList.length; i++) {
			//outputLog("Number is " + (requestList[i].byteLengthWithFill + 4 + packetReplyLength));
			if (requestList[i].byteLengthWithFill + 12 + 4 + packetWriteLength > self.maxPDU) { // 12 byte header for each item and 4 bytes for the data header
				if (numItems === 0) {
					outputLog("breaking when we shouldn't, byte length with fill is  " + requestList[i].byteLengthWithFill + " max byte request " + maxByteRequest, 0, self.connectionID);
					throw new Error("Somehow write request didn't split properly - exiting.  Report this as a bug.");
				}
				break;  // We can't fit this packet in here.
			}
			requestNumber++;
			numItems++;
			packetWriteLength += (requestList[i].byteLengthWithFill + 12 + 4); // Don't forget each request has a 12 byte header as well.
			//outputLog('I is ' + i + ' Addr Type is ' + requestList[i].addrtype + ' and type is ' + requestList[i].datatype + ' and DBNO is ' + requestList[i].dbNumber + ' and offset is ' + requestList[i].offset + ' bit ' + requestList[i].bitOffset + ' len ' + requestList[i].arrayLength);
			//S7AddrToBuffer(requestList[i]).copy(self.writeReq, 19 + numItems * 12);  // i or numItems?  used to be i.
			//itemBuffer = bufferizeS7Packet(requestList[i]);
			//itemBuffer.copy(dataBuffer, dataBufferPointer);
			//dataBufferPointer += itemBuffer.length;
			self.writePacketArray[thisPacketNumber].itemList.push(requestList[i]);
		}
		//		dataBuffer.copy(self.writeReq, 19 + (numItems + 1) * 12, 0, dataBufferPointer - 1);
	}
}
```

## prepareReadPacket

```js
NodeS7.prototype.prepareReadPacket = function() {
	var self = this, i;
	/*Зауважте, що для розміру PDU 240 НАЙБІЛЬША байт, яку ми можемо запросити, 
     залежить від кількості елементів.
	Щоб зрозуміти це, припустимо пакет розміром 247 байт: 
    - 7 Заголовок TPKT+COTP не враховується для PDU, тому 240 байт "S7 data" 
    - У відповіді ви ЗАВЖДИ маєте 12-байтовий заголовок S7.
    - Далі у вас є 2-байтовий заголовок параметра.
    - Далі у вас є 4-байтовий "item header" НА ITEM.
    Таким чином, у вас є:
    - накладні витрати 18 байтів для одного елемента, 
    - 22 байти для двох елементів, 
    - 26 байтів для 3-х і так далі. 
    Так, наприклад, ви можете запросити 240 - 22 = 218 байтів для двох елементів.
    Ми можемо обчислити максимальну довжину в байтах для одного запиту 
    як 4*Math.floor((self.maxPDU - 18)/4), щоб гарантувати, що ми не перетинаємо межі.
    */
	// Елементи – це фактичні елементи, які запитує користувач
	var itemList = self.polledReadBlockList;
    // Список запитів складається зі списку блоків, 
    // розділеного на фрагменти, доступні для читання PDU.
	var requestList = [];						

	// Validity check.
	for (i = itemList.length - 1; i >= 0; i--) {
		if (itemList[i] === undefined) {
			itemList.splice(i, 1);
			outputLog("Відкидання невизначеного елемента запиту.", 0, self.connectionID);
		}
	}

	// Sort the items using the sort function, by type and offset.
	itemList.sort(itemListSorter);

	// Just exit if there are no items.
	if (itemList.length === 0) {
		return undefined;
	}

	self.globalReadBlockList = [];

	// ...because you have to start your optimization somewhere.
	self.globalReadBlockList[0] = itemList[0];
	self.globalReadBlockList[0].itemReference = [];
	self.globalReadBlockList[0].itemReference.push(itemList[0]);

	var thisBlock = 0;
	itemList[0].block = thisBlock;
	var maxByteRequest = 4 * Math.floor((self.maxPDU - 18) / 4);  // Absolutely must not break a real array into two requests.  Maybe we can extend by two bytes when not DINT/REAL/INT.

	// Optimize the items into blocks
	for (i = 1; i < itemList.length; i++) {
		// Skip T, C, P types
		if ((itemList[i].areaS7Code !== self.globalReadBlockList[thisBlock].areaS7Code) ||   	// Can't optimize between areas
			(itemList[i].dbNumber !== self.globalReadBlockList[thisBlock].dbNumber) ||			// Can't optimize across DBs
			(!self.isOptimizableArea(itemList[i].areaS7Code)) || 					// Can't optimize T,C (I don't think) and definitely not P.
			((itemList[i].offset - self.globalReadBlockList[thisBlock].offset + itemList[i].byteLength) > maxByteRequest) ||      	// If this request puts us over our max byte length, create a new block for consistency reasons.
			(itemList[i].offset - (self.globalReadBlockList[thisBlock].offset + self.globalReadBlockList[thisBlock].byteLength) > self.maxGap)) {		// If our gap is large, create a new block.

			outputLog("Skipping optimization of item " + itemList[i].addr, 0, self.connectionID);

			// At this point we give up and create a new block.
			thisBlock = thisBlock + 1;
			self.globalReadBlockList[thisBlock] = itemList[i]; // By reference.
			//				itemList[i].block = thisBlock; // Don't need to do this.
			self.globalReadBlockList[thisBlock].isOptimized = false;
			self.globalReadBlockList[thisBlock].itemReference = [];
			self.globalReadBlockList[thisBlock].itemReference.push(itemList[i]);
		} else {
			outputLog("Attempting optimization of item " + itemList[i].addr + " with " + self.globalReadBlockList[thisBlock].addr, 0, self.connectionID);
			// This next line checks the maximum.
			// Think of this situation - we have a large request of 40 bytes starting at byte 10.
			//	Then someone else wants one byte starting at byte 12.  The block length doesn't change.
			//
			// But if we had 40 bytes starting at byte 10 (which gives us byte 10-49) and we want byte 50, our byte length is 50-10 + 1 = 41.
			self.globalReadBlockList[thisBlock].byteLength = Math.max(self.globalReadBlockList[thisBlock].byteLength, itemList[i].offset - self.globalReadBlockList[thisBlock].offset + itemList[i].byteLength);

			// Point the buffers (byte and quality) to a sliced version of the optimized block.  This is by reference (same area of memory)
			itemList[i].byteBuffer = self.globalReadBlockList[thisBlock].byteBuffer.slice(itemList[i].offset - self.globalReadBlockList[thisBlock].offset, itemList[i].offset - self.globalReadBlockList[thisBlock].offset + itemList[i].byteLength);
			itemList[i].qualityBuffer = self.globalReadBlockList[thisBlock].qualityBuffer.slice(itemList[i].offset - self.globalReadBlockList[thisBlock].offset, itemList[i].offset - self.globalReadBlockList[thisBlock].offset + itemList[i].byteLength);

			// For now, change the request type here, and fill in some other things.

			// I am not sure we want to do these next two steps.
			// It seems like things get screwed up when we do this.
			// Since self.globalReadBlockList[thisBlock] exists already at this point, and our buffer is already set, let's not do this now.
			// self.globalReadBlockList[thisBlock].datatype = 'BYTE';
			// self.globalReadBlockList[thisBlock].dtypelen = 1;
			self.globalReadBlockList[thisBlock].isOptimized = true;
			self.globalReadBlockList[thisBlock].itemReference.push(itemList[i]);
		}
	}

	var thisRequest = 0;

	//	outputLog("Preparing the read packet...");

	// Split the blocks into requests, if they're too large.
	for (i = 0; i < self.globalReadBlockList.length; i++) {
		// Always create a request for a self.globalReadBlockList.
		requestList[thisRequest] = self.globalReadBlockList[i].clone();

		// How many parts?
		self.globalReadBlockList[i].parts = Math.ceil(self.globalReadBlockList[i].byteLength / maxByteRequest);
		outputLog("self.globalReadBlockList " + i + " parts is " + self.globalReadBlockList[i].parts + " offset is " + self.globalReadBlockList[i].offset + " MBR is " + maxByteRequest, 1, self.connectionID);
		var startByte = self.globalReadBlockList[i].offset;
		var remainingLength = self.globalReadBlockList[i].byteLength;

		self.globalReadBlockList[i].requestReference = [];

		// If we're optimized...
		for (var j = 0; j < self.globalReadBlockList[i].parts; j++) {
			requestList[thisRequest] = self.globalReadBlockList[i].clone();
			self.globalReadBlockList[i].requestReference.push(requestList[thisRequest]);
			//outputLog(self.globalReadBlockList[i]);
			//outputLog(self.globalReadBlockList.slice(i,i+1));
			requestList[thisRequest].offset = startByte;
			requestList[thisRequest].byteLength = Math.min(maxByteRequest, remainingLength);
			requestList[thisRequest].byteLengthWithFill = requestList[thisRequest].byteLength;
			if (requestList[thisRequest].byteLengthWithFill % 2) { requestList[thisRequest].byteLengthWithFill += 1; }
			// Just for now...
			if (self.globalReadBlockList[i].parts > 1) {
				requestList[thisRequest].datatype = 'BYTE';
				requestList[thisRequest].dtypelen = 1;
				requestList[thisRequest].arrayLength = requestList[thisRequest].byteLength;//self.globalReadBlockList[thisBlock].byteLength;
			}
			remainingLength -= maxByteRequest;
			thisRequest++;
			startByte += maxByteRequest;
		}
	}

	//requestList[5].offset = 243;
	//	requestList = self.globalReadBlockList;

	// The packetizer...
	var requestNumber = 0;

	self.clearReadPacketTimeouts();
	self.readPacketArray = [];

	while (requestNumber < requestList.length) {

		var numItems = 0;
		self.readReqHeader.copy(self.readReq, 0);

		// Packet's expected reply length
		var packetReplyLength = 12 + 2;  //
		var packetRequestLength = 12; //s7 header and parameter header

		self.readPacketArray.push(new S7Packet());
		var thisPacketNumber = self.readPacketArray.length - 1;
		// don't set a fixed sequence number here. Instead, set it just before sending to avoid conflict with write sequence numbers
		self.readPacketArray[thisPacketNumber].seqNum = 0;
		self.readPacketArray[thisPacketNumber].itemList = [];  // Initialize as array.

		for (i = requestNumber; i < requestList.length; i++) {
			//outputLog("Number is " + (requestList[i].byteLengthWithFill + 4 + packetReplyLength));
			if (requestList[i].byteLengthWithFill + 4 + packetReplyLength > self.maxPDU || packetRequestLength + 12 > self.maxPDU) {
				outputLog("Splitting request: " + numItems + " items, requestLength would be " + (packetRequestLength + 12) + ", replyLength would be " + (requestList[i].byteLengthWithFill + 4 + packetReplyLength) + ", PDU is " + self.maxPDU, 1, self.connectionID);
				if (numItems === 0) {
					outputLog("breaking when we shouldn't, rlibl " + requestList[i].byteLengthWithFill + " MBR " + maxByteRequest, 0, self.connectionID);
					throw new Error("Somehow write request didn't split properly - exiting.  Report this as a bug.");
				}
				break;  // We can't fit this packet in here.
			}
			requestNumber++;
			numItems++;
			packetReplyLength += (requestList[i].byteLengthWithFill + 4);
			packetRequestLength += 12;
			//outputLog('I is ' + i + ' Addr Type is ' + requestList[i].addrtype + ' and type is ' + requestList[i].datatype + ' and DBNO is ' + requestList[i].dbNumber + ' and offset is ' + requestList[i].offset + ' bit ' + requestList[i].bitOffset + ' len ' + requestList[i].arrayLength);
			// skip this for now S7AddrToBuffer(requestList[i]).copy(self.readReq, 19 + numItems * 12);  // i or numItems?
			self.readPacketArray[thisPacketNumber].itemList.push(requestList[i]);
		}
	}
	self.readPacketValid = true;
}
```

## sendReadPacket

```js
NodeS7.prototype.sendReadPacket = function() {
	var self = this, i, j, flagReconnect = false;

	outputLog("SendReadPacket called", 1, self.connectionID);

    // Викликати зворотний виклик, якщо нас запитують нульові теги - для узгодженості
	if (!self.readPacketArray.length && (typeof(self.readDoneCallback) === "function")) {
        // Дані є другим аргументом і не мають бути undefined
		self.readDoneCallback(false, {});  
	}

	for (i = 0; i < self.readPacketArray.length; i++) {
		if (self.readPacketArray[i].sent) { continue; }
		if (self.parallelJobsNow >= self.maxParallel) { continue; }

		// Set sequence number of packet here
		self.readPacketArray[i].seqNum = self.getNextSeqNum();

		// From here down is SENDING the packet
		self.readPacketArray[i].reqTime = process.hrtime();
		self.readReq.writeUInt8(self.readPacketArray[i].itemList.length, 18);
        // buffer length
		self.readReq.writeUInt16BE(19 + self.readPacketArray[i].itemList.length * 12, 2); 
		self.readReq.writeUInt16BE(self.readPacketArray[i].seqNum, 11);
        // Parameter length - 14 for one read, 28 for 2.
		self.readReq.writeUInt16BE(self.readPacketArray[i].itemList.length * 12 + 2, 13); 

		for (j = 0; j < self.readPacketArray[i].itemList.length; j++) {
			S7AddrToBuffer(self.readPacketArray[i].itemList[j], false).copy(self.readReq,
                                                                            19 + j * 12);
		}

		if (self.isoConnectionState == 4) {
			outputLog('Sending Read Packet With Sequence Number ' +
                      self.readPacketArray[i].seqNum, 1, self.connectionID);
			self.readPacketArray[i].timeout = setTimeout(function() {
				self.packetTimeout.apply(self, arguments);
			}, self.globalTimeout, "read", self.readPacketArray[i].seqNum);
			//відправлення даних
            self.isoclient.write(self.readReq.slice(0, 19 +
             	self.readPacketArray[i].itemList.length * 12));
			self.readPacketArray[i].sent = true;
			self.readPacketArray[i].rcvd = false;
			self.readPacketArray[i].timeoutError = false;
			self.parallelJobsNow += 1;
		} else {
			//outputLog('Somehow got into read block without proper 
            // self.isoConnectionState of 3.  Disconnect.');
			self.readPacketArray[i].sent = true;
			self.readPacketArray[i].rcvd = false;
			self.readPacketArray[i].timeoutError = true;
			if (!flagReconnect) {
				// Prevent duplicates
				outputLog('Not Sending Read Packet because we are not connected' +
                          '- ISO CS is ' 
                          + self.isoConnectionState, 0, self.connectionID);
			}
			// This is essentially an instantTimeout.
			if (self.isoConnectionState === 0) {
				flagReconnect = true;
			}
			outputLog('Запит PacketTimeout через ISO CS NOT 4 - READ SN ' 
                      + self.readPacketArray[i].seqNum, 1, self.connectionID);
			self.readPacketArray[i].timeout = setTimeout(function() {
				self.packetTimeout.apply(self, arguments);
			}, 0, "read", self.readPacketArray[i].seqNum);
		}
	}
}
```



## sendWritePacket

```js
NodeS7.prototype.sendWritePacket = function() {
	var self = this, i, dataBuffer, itemBuffer, dataBufferPointer, flagReconnect;

	dataBuffer = Buffer.alloc(8192);

	self.writeInQueue = false;

	for (i = 0; i < self.writePacketArray.length; i++) {
		if (self.writePacketArray[i].sent) { continue; }
		if (self.parallelJobsNow >= self.maxParallel) { continue; }
		// From here down is SENDING the packet
		self.writePacketArray[i].reqTime = process.hrtime();
		self.writeReq.writeUInt8(self.writePacketArray[i].itemList.length, 18);
		self.writeReq.writeUInt16BE(self.writePacketArray[i].seqNum, 11);

		dataBufferPointer = 0;
		for (var j = 0; j < self.writePacketArray[i].itemList.length; j++) {
			S7AddrToBuffer(self.writePacketArray[i].itemList[j], true).copy(self.writeReq, 19 + j * 12);
			itemBuffer = getWriteBuffer(self.writePacketArray[i].itemList[j]);
			itemBuffer.copy(dataBuffer, dataBufferPointer);
			dataBufferPointer += itemBuffer.length;
			// NOTE: It seems that when writing, the data that is sent must have a "fill byte" so that data length is even only for all
			//  but the last request.  The last request must have no padding.  So we add the padding here.
			if (j < (self.writePacketArray[i].itemList.length - 1)) {
				if (itemBuffer.length % 2) {
					dataBufferPointer += 1;
				}
			}
		}

		//		outputLog('DataBufferPointer is ' + dataBufferPointer);
		self.writeReq.writeUInt16BE(19 + self.writePacketArray[i].itemList.length * 12 + dataBufferPointer, 2); // buffer length
		self.writeReq.writeUInt16BE(self.writePacketArray[i].itemList.length * 12 + 2, 13); // Parameter length - 14 for one read, 28 for 2.
		self.writeReq.writeUInt16BE(dataBufferPointer, 15); // Data length - as appropriate.

		dataBuffer.copy(self.writeReq, 19 + self.writePacketArray[i].itemList.length * 12, 0, dataBufferPointer);

		if (self.isoConnectionState === 4) {
			//			outputLog('writing' + (19+dataBufferPointer+self.writePacketArray[i].itemList.length*12));
			self.writePacketArray[i].timeout = setTimeout(function() {
				self.packetTimeout.apply(self, arguments);
			}, self.globalTimeout, "write", self.writePacketArray[i].seqNum);
			self.isoclient.write(self.writeReq.slice(0, 19 + dataBufferPointer + self.writePacketArray[i].itemList.length * 12));  // was 31
			self.writePacketArray[i].sent = true;
			self.writePacketArray[i].rcvd = false;
			self.writePacketArray[i].timeoutError = false;
			self.parallelJobsNow += 1;
			outputLog('Sending Write Packet With Sequence Number ' + self.writePacketArray[i].seqNum, 1, self.connectionID);
		} else {
			//			outputLog('Somehow got into write block without proper isoConnectionState of 4.  Disconnect.');
			//			connectionReset();
			//			setTimeout(connectNow, 2000, connectionParams);
			// This is essentially an instantTimeout.
			self.writePacketArray[i].sent = true;
			self.writePacketArray[i].rcvd = false;
			self.writePacketArray[i].timeoutError = true;

			// Without the scopePlaceholder, this doesn't work.   writePacketArray[i] becomes undefined.
			// The reason is that the value i is part of a closure and when seen "nextTick" has the same value
			// it would have just after the FOR loop is done.
			// (The FOR statement will increment it to beyond the array, then exit after the condition fails)
			// scopePlaceholder works as the array is de-referenced NOW, not "nextTick".
//dm			var scopePlaceholder = self.writePacketArray[i].seqNum;
//dm			process.nextTick(function() {
//dm				self.packetTimeout("write", scopePlaceholder);
//dm			});

			self.writePacketArray[i].timeout = setTimeout(function () {
				self.packetTimeout.apply(self, arguments);
			}, 0, "write", self.writePacketArray[i].seqNum);

			if (self.isoConnectionState === 0) {
				flagReconnect = true;
			}
		}
	}
/* NOTE: We no longer do this here.
Reconnects are done on the response that we will get from the above packets.
Reason: We could have some packets waiting for timeout from the PLC, and others coming back instantly.	
	if (flagReconnect) {
		//		console.log("Asking for callback next tick and my ID is " + self.connectionID);
		clearTimeout(self.reconnectTimer);
		self.reconnectTimer = setTimeout(function() {
			//			console.log("Next tick is here and my ID is " + self.connectionID);
			outputLog("The scheduled reconnect from sendWritePacket is happening now", 1, self.connectionID);
			self.connectNow(self.connectionParams);  // We used to do this NOW - not NextTick() as we need to mark isoConnectionState as 1 right now.  Otherwise we queue up LOTS of connects and crash.
		}, 0);
	}*/
}
```

## onResponse

```js
NodeS7.prototype.onResponse = function(theData) {
	var self = this;
	// Packet Validity Check.  Note that this will pass even with a "not available" response received from the server.
	// For length calculation and verification:
	// data[4] = COTP header length. Normally 2.  This doesn't include the length byte so add 1.
	// read(13) is parameter length.  Normally 4.
	// read(14) is data length.  (Includes item headers)
	// 12 is length of "S7 header"
	// Then we need to add 4 for TPKT header.

	// Decrement our parallel jobs now

	// NOT SO FAST - can't do this here.  If we time out, then later get the reply, we can't decrement this twice.  Or the CPU will not like us.  Do it if not rcvd.  self.parallelJobsNow--;

	// hotfix for #78, prevents RangeErrors for undersized packets
	if (!(theData && theData.length > 6)) {
		outputLog('INVALID READ RESPONSE - DISCONNECTING');
		outputLog("The incoming packet doesn't have the required minimum length of 7 bytes");
		outputLog(theData);
		self.connectionReset();
		return;
	}

	var data=checkRFCData(theData);

	if(data==="fastACK"){
		//read again and wait for the requested data
		outputLog('Fast Acknowledge received.', 0, self.connectionID);
		self.isoclient.removeAllListeners('error');
		self.isoclient.removeAllListeners('data');
		self.isoclient.on('data', function() {
			self.onResponse.apply(self, arguments);
		});
		self.isoclient.on('error', function() {
			self.readWriteError.apply(self, arguments);
		});
	}else if( data[7] === 0x32 ){//check the validy of FA+S7 package

		//*********************   VALIDY CHECK ***********************************
		//TODO: Check S7-Header properly
		if (data.length > 8 && data[8] != 3) {
			outputLog('PDU type (byte 8) was returned as ' + data[8] + ' where the response PDU of 3 was expected.');
			outputLog('Maybe you are requesting more than 240 bytes of data in a packet?');
			outputLog(data);
			self.connectionReset();
			return null;
		}
		// The smallest read packet will pass a length check of 25.  For a 1-item write response with no data, length will be 22.
		if (data.length > data.readInt16BE(2)) {
			outputLog("An oversize packet was detected.  Excess length is " + (data.length - data.readInt16BE(2)) + ".  ");
			outputLog("We assume this is because two packets were sent at nearly the same time by the PLC.");
			outputLog("We are slicing the buffer and scheduling the second half for further processing next loop.");
			setTimeout(function() {
				self.onResponse.apply(self, arguments);
			}, 0, data.slice(data.readInt16BE(2)));  // This re-triggers this same function with the sliced-up buffer.
			// was used as a test		setTimeout(process.exit, 2000);
		}

		if (data.length < data.readInt16BE(2) || data.readInt16BE(2) < 22 || data[5] !== 0xf0 || data[4] + 1 + 12 + 4 + data.readInt16BE(13) + data.readInt16BE(15) !== data.readInt16BE(2) || !(data[6] >> 7) || (data[7] !== 0x32) || (data[8] !== 3)) {
			outputLog('INVALID READ RESPONSE - DISCONNECTING');
			outputLog('TPKT Length From Header is ' + data.readInt16BE(2) + ' and RCV buffer length is ' + data.length + ' and COTP length is ' + data.readUInt8(4) + ' and data[6] is ' + data[6]);
			outputLog(data);
			self.connectionReset();
			return null;
		}

		//**********************   GO ON  *************************
		// Log the receive
		outputLog('Received ' + data.readUInt16BE(15) + ' bytes of S7-data from PLC.  Sequence number is ' + data.readUInt16BE(11), 1, self.connectionID);

		// Check the sequence number
		var foundSeqNum; // self.readPacketArray.length - 1;
		var isReadResponse, isWriteResponse;

		//	for (packetCount = 0; packetCount < self.readPacketArray.length; packetCount++) {
		//		if (self.readPacketArray[packetCount].seqNum == data.readUInt16BE(11)) {
		//			foundSeqNum = packetCount;
		//			break;
		//		}
		//	}
		foundSeqNum = self.findReadIndexOfSeqNum(data.readUInt16BE(11));

		//	if (self.readPacketArray[packetCount] == undefined) {
		if (foundSeqNum === undefined) {
			foundSeqNum = self.findWriteIndexOfSeqNum(data.readUInt16BE(11));
			if (foundSeqNum !== undefined) {
				//		for (packetCount = 0; packetCount < self.writePacketArray.length; packetCount++) {
				//			if (self.writePacketArray[packetCount].seqNum == data.readUInt16BE(11)) {
				//				foundSeqNum = packetCount;
				self.writeResponse(data, foundSeqNum);
				isWriteResponse = true;
				//				break;
			}


		} else {
			isReadResponse = true;
			self.readResponse(data, foundSeqNum);
		}

		if ((!isReadResponse) && (!isWriteResponse)) {
			outputLog("Sequence number that arrived wasn't a write reply either - dropping");
			outputLog(data);
			// 	I guess this isn't a showstopper, just ignore it.
			//		self.isoclient.end();
			//		setTimeout(self.connectNow, 2000, self.connectionParams);
			return null;
		}

	}else{
		outputLog('INVALID READ RESPONSE - DISCONNECTING');
		outputLog('TPKT Length From Header is ' + theData.readInt16BE(2) + ' and RCV buffer length is ' + theData.length + ' and COTP length is ' + theData.readUInt8(4) + ' and data[6] is ' + theData[6]);
		outputLog(theData);
		self.connectionReset();
		return null;
	}

}
```

## findReadIndexOfSeqNum

```js
NodeS7.prototype.findReadIndexOfSeqNum = function(seqNum) {
	var self = this, packetCounter;
	for (packetCounter = 0; packetCounter < self.readPacketArray.length; packetCounter++) {
		if (self.readPacketArray[packetCounter].seqNum == seqNum) {
			return packetCounter;
		}
	}
	return undefined;
}
```

## findWriteIndexOfSeqNum

```js
NodeS7.prototype.findWriteIndexOfSeqNum = function(seqNum) {
	var self = this, packetCounter;
	for (packetCounter = 0; packetCounter < self.writePacketArray.length; packetCounter++) {
		if (self.writePacketArray[packetCounter].seqNum == seqNum) {
			return packetCounter;
		}
	}
	return undefined;
}
```

## writeResponse

```js
NodeS7.prototype.writeResponse = function(data, foundSeqNum) {
	var self = this, dataPointer = 21, i, anyBadQualities;

	for (var itemCount = 0; itemCount < self.writePacketArray[foundSeqNum].itemList.length; itemCount++) {
		//		outputLog('Pointer is ' + dataPointer);
		dataPointer = processS7WriteItem(data, self.writePacketArray[foundSeqNum].itemList[itemCount], dataPointer);
		if (!dataPointer) {
			outputLog('Stopping Processing Write Response Packet due to unrecoverable packet error');
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

## doneSending

```
function doneSending(element) {
	return ((element.sent && element.rcvd) ? true : false);
}
```

## readResponse

```js
NodeS7.prototype.readResponse = function(data, foundSeqNum) {
	var self = this, i;
	var anyBadQualities;
	var dataPointer = 21; // For non-routed packets we start at byte 21 of the packet.  If we do routing it will be more than this.
	var dataObject = {};

	//	if (self.readPacketArray.timeod (i forget what was going on here)
	//	if (typeof(data) === "undefined") {
	//		outputLog("Undefined " + foundSeqNum);
	//	} else {
	//		outputLog("Defined " + foundSeqNum);
	//	}

	outputLog("ReadResponse called", 1, self.connectionID);

	if (!self.readPacketArray[foundSeqNum].sent) {
		outputLog('WARNING: Received a read response packet that was not marked as sent', 0, self.connectionID);
		//TODO - fix the network unreachable error that made us do this
		return null;
	}

	if (self.readPacketArray[foundSeqNum].rcvd) {
		outputLog('WARNING: Received a read response packet that was already marked as received', 0, self.connectionID);
		return null;
	}

	for (var itemCount = 0; itemCount < self.readPacketArray[foundSeqNum].itemList.length; itemCount++) {
		dataPointer = processS7Packet(data, self.readPacketArray[foundSeqNum].itemList[itemCount], dataPointer, self.connectionID);
		if (!dataPointer) {
			outputLog('Received a ZERO RESPONSE Processing Read Packet due to unrecoverable packet error', 0, self.connectionID);
			// We rely on this for our timeout.
		}
	}

	// Make a note of the time it took the PLC to process the request.
	self.readPacketArray[foundSeqNum].reqTime = process.hrtime(self.readPacketArray[foundSeqNum].reqTime);
	outputLog('Time is ' + self.readPacketArray[foundSeqNum].reqTime[0] + ' seconds and ' + Math.round(self.readPacketArray[foundSeqNum].reqTime[1] * 10 / 1e6) / 10 + ' ms.', 1, self.connectionID);

	// Do the bookkeeping for packet and timeout.
	if (!self.readPacketArray[foundSeqNum].rcvd) {
		self.readPacketArray[foundSeqNum].rcvd = true;
		self.parallelJobsNow--;
	}
	clearTimeout(self.readPacketArray[foundSeqNum].timeout);

	if (self.readPacketArray.every(doneSending)) {  // if sendReadPacket returns true we're all done.
		// Mark our packets unread for next time.
		for (i = 0; i < self.readPacketArray.length; i++) {
			self.readPacketArray[i].sent = false;
			self.readPacketArray[i].rcvd = false;
		}

		anyBadQualities = false;

		// Loop through the global block list...
		for (i = 0; i < self.globalReadBlockList.length; i++) {
			var lengthOffset = 0;
			// For each block, we loop through all the requests.  Remember, for all but large arrays, there will only be one.
			for (var j = 0; j < self.globalReadBlockList[i].requestReference.length; j++) {
				// Now that our request is complete, we reassemble the BLOCK byte buffer as a copy of each and every request byte buffer.
				self.globalReadBlockList[i].requestReference[j].byteBuffer.copy(self.globalReadBlockList[i].byteBuffer, lengthOffset, 0, self.globalReadBlockList[i].requestReference[j].byteLength);
				self.globalReadBlockList[i].requestReference[j].qualityBuffer.copy(self.globalReadBlockList[i].qualityBuffer, lengthOffset, 0, self.globalReadBlockList[i].requestReference[j].byteLength);
				lengthOffset += self.globalReadBlockList[i].requestReference[j].byteLength;
			}
			// For each ITEM reference pointed to by the block, we process the item.
			for (var k = 0; k < self.globalReadBlockList[i].itemReference.length; k++) {
				processS7ReadItem(self.globalReadBlockList[i].itemReference[k]);
				outputLog('Address ' + self.globalReadBlockList[i].itemReference[k].addr + ' has value ' + self.globalReadBlockList[i].itemReference[k].value + ' and quality ' + self.globalReadBlockList[i].itemReference[k].quality, 1, self.connectionID);
				if (!isQualityOK(self.globalReadBlockList[i].itemReference[k].quality)) {
					anyBadQualities = true;
					dataObject[self.globalReadBlockList[i].itemReference[k].useraddr] = self.globalReadBlockList[i].itemReference[k].quality;
				} else {
					dataObject[self.globalReadBlockList[i].itemReference[k].useraddr] = self.globalReadBlockList[i].itemReference[k].value;
				}
			}
		}

// Not as of Feb 2019		if (self.resetPending) {
// Not as of Feb 2019			self.resetNow();
// Not as of Feb 2019		}

		if (!self.writeInQueue) {
			if (self.resetPending) {
				outputLog('Calling reset from readResponse as there is one pending',0,self.connectionID);
				self.resetNow();
			}
			if (self.isoConnectionState === 0) {
				self.connectNow(self.connectionParams, false);
			}
		} else {
			outputLog('Write In Queue.  ICS ' + self.isoConnectionState + ' resetPending ' + self.resetPending,1,self.connectionID);		
		}

		// Inform our user that we are done and that the values are ready for pickup.
		outputLog("We are calling back our readDoneCallback.", 1, self.connectionID);
		if (typeof (self.readDoneCallback) === 'function') {
			self.readDoneCallback(anyBadQualities, dataObject);
		}

		if (!self.isReading() && self.writeInQueue) {
			outputLog("SendWritePacket called because write was queued.", 0, self.connectionID);
			self.sendWritePacket();
		}
	} else {
		self.sendReadPacket();
	}
}
```

## checkRFCData

```js
function checkRFCData(data){
   var ret=null;
   var RFC_Version = data[0];
   var TPKT_Length = data.readInt16BE(2);
   var TPDU_Code = data[5]; //Data==0xF0 !!
   var LastDataUnit = data[6];//empty fragmented frame => 0=not the last package; 1=last package

   if(RFC_Version !==0x03 && TPDU_Code !== 0xf0){
      //Check if its an RFC package and a Data package
      return 'error';
   }else if((LastDataUnit >> 7) === 0 && TPKT_Length == data.length &&  data.length === 7){
      // Check if its a Fast Acknowledge package from older PLCs or  WinAC or data is too long ...
      // For example: <Buffer 03 00 00 07 02 f0 00> => data.length==7
      ret='fastACK';
   }else if((LastDataUnit >> 7) == 1 && TPKT_Length <= data.length){
      // Check if its an  FastAcknowledge package + S7Data package
      // <Buffer 03 00 00 1b 02 f0 80 32 03 00 00 00 00 00 08 00 00 00 00 f0 00 00 01 00 01 00 f0> => data.length==7+20=27
      ret=data;
   }else if((LastDataUnit >> 7) == 0  && TPKT_Length !== data.length){
      // Check if its an  FastAcknowledge package + FastAcknowledge package+ S7Data package
      // Possibly because NodeS7 or Application is too slow at this moment!
      // <Buffer 03 00 00 07 02 f0 00 03 00 00 1b 02 f0 80 32 03 00 00 00 00 00 08 00 00 00 00 f0 00 00 01 00 01 00 f0>  => data.length==7+7+20=34
      ret=data.slice(7, data.length)//Cut off the first Fast Acknowledge Packet
   }else{
      ret='error';
   }
   return ret;
}
```

## S7Packet

```js
function S7Packet() {
	this.seqNum = undefined;				// Made-up sequence number to watch for.
	this.itemList = undefined;  			// This will be assigned the object that details what was in the request.
	this.reqTime = undefined;
	this.sent = false;						// Have we sent the packet yet?
	this.rcvd = false;						// Are we waiting on a reply?
	this.timeoutError = undefined;			// The packet is marked with error on timeout so we don't then later switch to good data.
	this.timeout = undefined;				// The timeout for use with clearTimeout()
}
```

## isQualityOK

```js
function isQualityOK(obj) {
	if (typeof obj === "string") {
		if (obj !== 'OK') { return false; }
	} else if (Array.isArray(obj)) {
		for (var i = 0; i < obj.length; i++) {
			if (typeof obj[i] !== "string" || obj[i] !== 'OK') { return false; }
		}
	}
	return true;
}
```

## itemListSorter

```js
function itemListSorter(a, b) {
	// Feel free to manipulate these next two lines...
	if (a.areaS7Code < b.areaS7Code) { return -1; }
	if (a.areaS7Code > b.areaS7Code) { return 1; }

	// Group first the items of the same DB
	if (a.addrtype === 'DB') {
		if (a.dbNumber < b.dbNumber) { return -1; }
		if (a.dbNumber > b.dbNumber) { return 1; }
	}

	// But for byte offset we need to start at 0.
	if (a.offset < b.offset) { return -1; }
	if (a.offset > b.offset) { return 1; }

	// Then bit offset
	if (a.bitOffset < b.bitOffset) { return -1; }
	if (a.bitOffset > b.bitOffset) { return 1; }

	// Then item length - most first.  This way smaller items are optimized into bigger ones if they have the same starting value.
	if (a.byteLength > b.byteLength) { return -1; }
	if (a.byteLength < b.byteLength) { return 1; }
}
```
