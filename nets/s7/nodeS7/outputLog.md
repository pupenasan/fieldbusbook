[<-- До опису бібліотеки](README.md) 

# outputLog

Виводить в консоль повідомлення `txt` з вказаним `id` та рівнем критичності `debugLevel`

```js
function outputLog(txt, debugLevel, id) {
	if (silentMode) return;

	var idtext;
	if (typeof (id) === 'undefined') {
		idtext = '';
	} else {
		idtext = ' ' + id;
	}
	if (typeof (debugLevel) === 'undefined' || effectiveDebugLevel >= debugLevel) {
		console.log('[' + process.hrtime() + idtext + '] ' + util.format(txt));
	}
}
```



[<-- До опису бібліотеки](README.md) 





