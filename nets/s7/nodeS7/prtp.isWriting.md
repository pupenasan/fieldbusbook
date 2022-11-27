[<-- До опису бібліотеки](README.md) 

# isWriting

Показує чи бібліотека в процесі запису даних. Для цього перебирає усі елементи у `writePacketArray` і якщо якийсь має властивість `sent === true` повертає `true`

```js
NodeS7.prototype.isWriting = function() {
	var self = this, i;
	// Walk through the array and if any packets are marked as sent, it means we haven't received our final confirmation.
	for (i = 0; i < self.writePacketArray.length; i++) {
		if (self.writePacketArray[i].sent === true) { return true }
	}
	return false;
}
```

[<-- До опису бібліотеки](README.md) 





