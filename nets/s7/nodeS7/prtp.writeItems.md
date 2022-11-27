[<-- До опису бібліотеки](README.md) 

# writeItems

Формує масив `instantWriteBlockList` з елементів типу `S7Item` що записуються та їх значень після чого викликає `prepareWritePacket` для ...  та `sendWritePacket` для.

```js
writeItems (arg, value, cb)
```

`arg` - назва змінної, або масив назв для запису 

`value` - значення для запису, або масив значень для запису

`cb` - функція зворотного виклику для виклику після запису

```js
conn.writeItems('TEST3', true, valuesWritten);
conn.writeItems(['TEST5', 'TEST6'], [ 867.5309, 9 ], valuesWritten); 
conn.writeItems('TEST7', [666, 777], valuesWritten); 
```

Функція робить наступне:

- перевіряє чи відбувається записування через функцію [isWriting()](prtp.isWriting.md) та `writeInQueue`, якщо так, то виходить
- Ставить у якості `writeDoneCallback` функцію `cb`
- Добавляє в масив `instantWriteBlockList` елементи через [stringToS7Addr](stringToS7Addr.md) відповідно до вхідних аргументів
- Викликає [prepareWritePacket](prtp.prepareWritePacket.md)

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

	// Перевірка валідності.
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





