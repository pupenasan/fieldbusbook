[<-- До опису бібліотеки](README.md) 

# setTranslationCB

```js
setTranslationCB (cb)
```

Встановлює функцію зворотного виклику `cb` при перетворенні.  Приклад: 

```js
var variables = {
      TEST1: 'MR4',          // Memory real at MD4
      TEST2: 'M32.2',        // Bit at M32.2
      TEST3: 'M20.0',        // Bit at M20.0
      TEST4: 'DB1,REAL0.20', // Array of 20 values in DB1
      TEST5: 'DB1,REAL4',    // Single real value
      TEST6: 'DB1,REAL8',    // Another single real value
      TEST7: 'DB1,INT12.2',  // Two integer value array
      TEST8: 'DB1,LREAL4',   // Single 8-byte real value
      TEST9: 'DB1,X14.0',    // Single bit in a data block
      TEST10: 'DB1,X14.0.8'  // Array of 8 bits in a data block
};
conn.setTranslationCB(function(tag) { return variables[tag]; }); 
```



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





