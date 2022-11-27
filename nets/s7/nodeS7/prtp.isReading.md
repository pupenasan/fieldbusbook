[<-- До опису бібліотеки](README.md) 

## isReading

Показує чи бібліотека в процесі читання даних

```js
NodeS7.prototype.isReading = function() {
	var self = this, i;
	// Пройдіться по масиву, і якщо якісь пакети позначені як надіслані, 
    // це означає, що ми не отримали остаточне підтвердження. 
	for (i = 0; i < self.readPacketArray.length; i++) {
		if (self.readPacketArray[i].sent === true) { return true }
	}
	return false;
}
```





[<-- До опису бібліотеки](README.md) 





