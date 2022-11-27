[<-- До опису бібліотеки](README.md) 

# itemListSorter

Сортує елементи масиву з S7Item по пріоритетності:

- areaS7Code
- addrtype
- offset
- bitOffset
- byteLength

Потрібне для формування 

```js
itemListSorter (a, b)
```



```js
function itemListSorter(a, b) {
	// Feel free to manipulate these next two lines...
	if (a.areaS7Code < b.areaS7Code) { return -1; }
	if (a.areaS7Code > b.areaS7Code) { return 1; }

	// Group first the items of the same DB
	if (a.addrtype === 'DB') {
		if (a.dbNumber < b.dbNumber) { return -1; }
		if (a.dbNumber > b.dbNumber) { return 1; }
	}

	// But for byte offset we need to start at 0.
	if (a.offset < b.offset) { return -1; }
	if (a.offset > b.offset) { return 1; }

	// Then bit offset
	if (a.bitOffset < b.bitOffset) { return -1; }
	if (a.bitOffset > b.bitOffset) { return 1; }

	// Then item length - most first.  This way smaller items are optimized into bigger ones if they have the same starting value.
	if (a.byteLength > b.byteLength) { return -1; }
	if (a.byteLength < b.byteLength) { return 1; }
}
```







[<-- До опису бібліотеки](README.md) 





