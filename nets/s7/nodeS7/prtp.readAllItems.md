[<-- До опису бібліотеки](README.md) 

```js
NodeS7.prototype.readAllItems = function(arg) {
	var self = this;
	outputLog("Читання всіх елементів (викликано readAllItems)", 1, self.connectionID);
	if (typeof arg === "function") {
		self.readDoneCallback = arg;
	} else {
		self.readDoneCallback = doNothing;
	}

	if (self.isoConnectionState !== 4) {
		outputLog("Неможливо прочитати без підключення. Повернути неправильні значення."
                  , 0, self.connectionID);
	} // Для кращої роботи під час автоматичного повторного підключення - не повертати зараз 

	// Перевірте, чи ВСЕ виконано... Ви можете подумати, 
    //що ми можемо розглянути паралельні завдання, 
    //і здебільшого ми можемо, але якщо один щойно закінчив, і ми опиняємось тут 
    //перш ніж почати іншу, це погано. 
	if (self.isWaiting()) {
		outputLog("Очікування читання для завершення всіх операцій R/W. " + 
                  "Повторно запустить readAllItems через 100 мс.", 0, self.connectionID);
		clearTimeout(self.rereadTimer);
		self.rereadTimer = setTimeout(function() {
			self.rereadTimer = undefined; //вже вистрілили, можете сміливо викидати
			self.readAllItems.apply(self, arguments);
		}, 100, arg);
		return;
	}

	// Now we check the array of adding and removing things.  
    // Only now is it really safe to do this.
	self.addRemoveArray.forEach(function(element) {
		outputLog('Adding or Removing ' + util.format(element), 1, self.connectionID);
		if (element.action === 'remove') {
			self.removeItemsNow(element.arg);
		}
		if (element.action === 'add') {
			self.addItemsNow(element.arg);
		}
	});

	self.addRemoveArray = []; // Clear for next time.

	if (!self.readPacketValid) { self.prepareReadPacket(); }

	// ideally...  incrementSequenceNumbers();

	outputLog("Calling SRP from RAI", 1, self.connectionID);
    // Note this sends the first few read packets depending 
    // on parallel connection restrictions.
	self.sendReadPacket(); 
}
```





[<-- До опису бібліотеки](README.md) 





