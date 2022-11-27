[<-- До опису бібліотеки](README.md) 

# prepareWritePacket

ToDo

Бере список блоків `instantWriteBlockList` та робить з нього `writePacketArray`. 

- `itemList = instantWriteBlockList`
- Створює `globalWriteBlockList` 
- `globalWriteBlockList[0] = itemList[0]`
- з одним 0-м елементом



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

	// На цей раз ми не робимо оптимізацію. 
    // Причина в цьому те що може спричинити 
	// Наприклад якщо ми записуєм M0.1 і M0.2 то оптимізація б спричниа запис MB0, 
    // який би також записував 0.0, 0.3, 0.4...
	// Так само зі змінними MW0 і MW2, записався б весь блок
	// Але якщо ви дуже, дуже хочете програму для того щоб це зробити, 
    // запишіть самі масив 
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
			self.writePacketArray[thisPacketNumber].itemList.push(requestList[i]);
		}
	}
}
```





[<-- До опису бібліотеки](README.md) 





