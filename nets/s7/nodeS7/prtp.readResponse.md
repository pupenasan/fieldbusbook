[<-- До опису бібліотеки](README.md) 

# readResponse

```js
readResponse (data, foundSeqNum)
```

Обробляє отриманий пакет `data` з номером `foundSeqNum` . 

```js
NodeS7.prototype.readResponse = function(data, foundSeqNum) {
	var self = this, i;
	var anyBadQualities;
    // Для немаршрутизованих пакетів ми починаємо з байта 21 пакету. 
    // Якщо ми зробимо маршрутизацію, це буде більше, ніж це.
    // dataPointer - зсув байта для Item 
	var dataPointer = 21; 
	var dataObject = {};

	outputLog("ReadResponse called", 1, self.connectionID);

	if (!self.readPacketArray[foundSeqNum].sent) {
		outputLog('ПОПЕРЕДЖЕННЯ. Отримано пакет відповіді на прочитання,' + 
                  ' який не було позначено як надісланий', 0, self.connectionID);
		return null;
	}

	if (self.readPacketArray[foundSeqNum].rcvd) {
		outputLog('ПОПЕРЕДЖЕННЯ. Отримано пакет відповіді на прочитання,' + 
                  ' який уже позначено як отриманий', 0, self.connectionID);
		return null;
	}

	for (var itemCount = 0; 
        itemCount < self.readPacketArray[foundSeqNum].itemList.length; itemCount++) {
		dataPointer = processS7Packet(data,
        self.readPacketArray[foundSeqNum].itemList[itemCount],
        dataPointer, self.connectionID);
		if (!dataPointer) {
			outputLog('Отримано НУЛЬОВУ ВІДПОВІДЬ Обробка пакета читання' +
                      ' через невиправну помилку пакета', 0, self.connectionID);
		}
	}

	// Записати час, який знадобився ПЛК для обробки запиту. 
	self.readPacketArray[foundSeqNum].reqTime =
        process.hrtime(self.readPacketArray[foundSeqNum].reqTime);
	outputLog('Time is ' + self.readPacketArray[foundSeqNum].reqTime[0] + ' seconds and '
              + Math.round(self.readPacketArray[foundSeqNum].reqTime[1] * 10 / 1e6) / 10
              + ' ms.', 1, self.connectionID);

	// Ведіть облік для пакетів і тайм-ауту.
	if (!self.readPacketArray[foundSeqNum].rcvd) {
		self.readPacketArray[foundSeqNum].rcvd = true;
		self.parallelJobsNow--;
	}
	clearTimeout(self.readPacketArray[foundSeqNum].timeout);
    
    //якщо sendReadPacket повертає true, все готово.
	if (self.readPacketArray.every(doneSending)) {  
		// Позначайте наші пакети непрочитаними для наступного разу.
		for (i = 0; i < self.readPacketArray.length; i++) {
			self.readPacketArray[i].sent = false;
			self.readPacketArray[i].rcvd = false;
		}
		anyBadQualities = false;
		// Переглядаємо глобальний список блоків ...
		for (i = 0; i < self.globalReadBlockList.length; i++) {
			var lengthOffset = 0;
			// Для кожного блоку ми переглядаємо всі запити. 
            // Пам’ятайте, що для всіх масивів, крім великих, буде лише один.
			for (var j = 0; j<self.globalReadBlockList[i].requestReference.length; j++) {
				// Тепер, коли наш запит завершено, ми знову збираємо байтовий буфер
                // BLOCK як копію кожного байтового буфера запиту. 
				self.globalReadBlockList[i].requestReference[j].byteBuffer.copy(
					self.globalReadBlockList[i].byteBuffer, lengthOffset, 0, 								self.globalReadBlockList[i].requestReference[j].byteLength);

				self.globalReadBlockList[i].requestReference[j].qualityBuffer.copy(
					self.globalReadBlockList[i].qualityBuffer, lengthOffset, 0, 		
                    self.globalReadBlockList[i].requestReference[j].byteLength);

                lengthOffset += self.globalReadBlockList[i].
                								requestReference[j].byteLength;
			}
			// Для кожного посилання ITEM, на яке вказує блок, ми обробляємо елемент.
			for (var k = 0; k < self.globalReadBlockList[i].itemReference.length; k++) {
				processS7ReadItem(self.globalReadBlockList[i].itemReference[k]);
				outputLog('Address ' 
                          + self.globalReadBlockList[i].itemReference[k].addr 
                          + ' has value ' + 
                          self.globalReadBlockList[i].itemReference[k].value 
                          + ' and quality ' 
                          + self.globalReadBlockList[i].itemReference[k].quality, 
                          1, self.connectionID);
				if (!isQualityOK(self.globalReadBlockList[i].itemReference[k].quality)) {
					anyBadQualities = true;
					dataObject[self.globalReadBlockList[i].itemReference[k].useraddr] 
                        = self.globalReadBlockList[i].itemReference[k].quality;
				} else {
					dataObject[self.globalReadBlockList[i].itemReference[k].useraddr] 
                        = self.globalReadBlockList[i].itemReference[k].value;
				}
			}
		}
		if (!self.writeInQueue) {
			if (self.resetPending) {
				outputLog('Скидання виклику з readResponse, оскільки є один',
                          0,self.connectionID);
				self.resetNow();
			}
			if (self.isoConnectionState === 0) {
				self.connectNow(self.connectionParams, false);
			}
		} else {
			outputLog('Write In Queue.  ICS ' 
                      + self.isoConnectionState 
                      + ' resetPending ' 
                      + self.resetPending,1,self.connectionID);		
		}

		// Inform our user that we are done and that the values are ready for pickup.
		outputLog("We are calling back our readDoneCallback.", 1, self.connectionID);
		if (typeof (self.readDoneCallback) === 'function') {
			self.readDoneCallback(anyBadQualities, dataObject);
		}

		if (!self.isReading() && self.writeInQueue) {
			outputLog("Викликає SendWritePacket оскільки була черга на запис.",
                      0, self.connectionID);
			self.sendWritePacket();
		}
	} else {
		self.sendReadPacket();
	}
}
```





[<-- До опису бібліотеки](README.md) 





