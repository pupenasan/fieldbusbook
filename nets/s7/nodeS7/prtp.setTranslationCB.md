[<-- До опису бібліотеки](README.md) 

## setTranslationCB

```js
NodeS7.prototype.setTranslationCB = function(cb) {
	var self = this;
	if (typeof cb === "function") {
		outputLog('Translation OK');
		self.translationCB = cb;
	}
}
```





[<-- До опису бібліотеки](README.md) 





