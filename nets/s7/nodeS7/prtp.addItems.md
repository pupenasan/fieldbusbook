[<-- До опису бібліотеки](README.md) 

# addItems

```js
addItems (arg) 
```

Добавляє в масив `addRemoveArray` елемент `arg`, приклад

```js
conn.addItems(['TEST1', 'TEST4']);
```



```js
NodeS7.prototype.addItems = function(arg) {
	var self = this;
	self.addRemoveArray.push({ arg: arg, action: 'add' });
}
```





[<-- До опису бібліотеки](README.md) 





