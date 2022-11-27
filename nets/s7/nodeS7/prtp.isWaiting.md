[<-- До опису бібліотеки](README.md) 

# isWaiting

 показує чи бібліотека зайнята читанням чи записом

```js
NodeS7.prototype.isWaiting = function() {
	var self = this;
	return (self.isReading() || self.isWriting());
}
```



[<-- До опису бібліотеки](README.md) 





