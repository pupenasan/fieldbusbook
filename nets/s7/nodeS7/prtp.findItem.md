[<-- До опису бібліотеки](README.md) 

## findItem

```js
NodeS7.prototype.findItem = function(useraddr) {
	var self = this, i;
	var commstate = { value: self.isoConnectionState !== 4, quality: 'OK' };
	if (useraddr === '_COMMERR') { return commstate; }
	for (i = 0; i < self.polledReadBlockList.length; i++) {
		if (self.polledReadBlockList[i].useraddr === useraddr) 
        { return self.polledReadBlockList[i]; }
	}
	return undefined;
}
```





[<-- До опису бібліотеки](README.md) 





