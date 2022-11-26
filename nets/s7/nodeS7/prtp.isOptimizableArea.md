[<-- До опису бібліотеки](README.md) 

## isOptimizableArea

```js
NodeS7.prototype.isOptimizableArea = function(area) {
	var self = this;

	if (self.doNotOptimize) { return false; } // Are we skipping all optimization due to user request?
	switch (area) {
		case 0x84: // db
		case 0x81: // input bytes
		case 0x82: // output bytes
		case 0x83: // memory bytes
			return true;
		default:
			return false;
	}
}
```





[<-- До опису бібліотеки](README.md) 





