[<-- До опису бібліотеки](README.md) 

```js
function readDT(buffer, offset, isUTC) {
	let year = fromBCD(buffer.readUInt8(offset));
	let month = fromBCD(buffer.readUInt8(offset + 1));
	let day = fromBCD(buffer.readUInt8(offset + 2));
	let hour = fromBCD(buffer.readUInt8(offset + 3));
	let min = fromBCD(buffer.readUInt8(offset + 4));
	let sec = fromBCD(buffer.readUInt8(offset + 5));
	let ms_1 = fromBCD(buffer.readUInt8(offset + 6));
	let ms_2 = fromBCD(buffer.readUInt8(offset + 7) & 0xf0);

	let date;
	if (isUTC) {
		date = new Date(Date.UTC((year > 89 ? 1900 : 2000) + year, month - 1,
			day, hour, min, sec, (ms_1 * 10) + (ms_2 / 10)))
	} else {
		date = new Date((year > 89 ? 1900 : 2000) + year, month - 1,
			day, hour, min, sec, (ms_1 * 10) + (ms_2 / 10));
	}

	return date;
}
```





[<-- До опису бібліотеки](README.md) 





