[<-- До опису бібліотеки](README.md) 

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





[<-- До опису бібліотеки](README.md) 





