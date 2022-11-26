[<-- До опису бібліотеки](README.md) 

## writeItems

```js
NodeS7.prototype.writeItems = function(arg, value, cb) {
	var self = this, i;
	outputLog("Preparing to WRITE " + arg + " to value " + value, 0, self.connectionID);
	if (self.isWriting() || self.writeInQueue) {
		outputLog("Ви повинні зачекати, доки завершаться всі попередні записи," + 
                  "перш ніж запланувати інший." , 0, self.connectionID);
		return 1;  // Слідкуйте за цим у своєму коді - 
        // 1 означає, що він фактично не входив до черги.
	}

	if (typeof cb === "function") {
		self.writeDoneCallback = cb;
	} else {
		self.writeDoneCallback = doNothing;
	}

	self.instantWriteBlockList = []; // Ініціалізація масиву.

	if (typeof arg === "string") {
		self.instantWriteBlockList.push(stringToS7Addr(self.translationCB(arg), 
                                                       arg, self.connectionParams));
		if (typeof (self.instantWriteBlockList[self.instantWriteBlockList.length - 1])
            !== "undefined") {
			self.instantWriteBlockList[self.instantWriteBlockList.length - 1]
                .writeValue = value;
		}
	} else if (Array.isArray(arg) && Array.isArray(value) 
               && (arg.length == value.length)) {
		for (i = 0; i < arg.length; i++) {
			if (typeof arg[i] === "string") {
				self.instantWriteBlockList.push(
                    stringToS7Addr(self.translationCB(arg[i]), arg[i],
                                   self.connectionParams));
				if (typeof (self.instantWriteBlockList[
                    self.instantWriteBlockList.length - 1]) !== "undefined") {
					self.instantWriteBlockList[self.instantWriteBlockList.length 
                                               - 1].writeValue = value[i];
				}
			}
		}
	}

	// Validity check.
	for (i = self.instantWriteBlockList.length - 1; i >= 0; i--) {
		if (self.instantWriteBlockList[i] === undefined) {
			self.instantWriteBlockList.splice(i, 1);
			outputLog("Скидання невизначеного елемента запису.");
		}
	}
	self.prepareWritePacket();
	if (!self.isReading()) {
		self.sendWritePacket();
	} else {
		if (self.writeInQueue) {
			outputLog("Запис уже був у черзі - слід запобігти вище",
                      1,self.connectionID);
		}
		self.writeInQueue = true;
		outputLog("Додавання запису в чергу");
	}
	return 0;
}
```





[<-- До опису бібліотеки](README.md) 





