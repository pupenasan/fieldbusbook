[<-- До опису бібліотеки](README.md) 

```js
function readDTL(buffer, offset, isUTC) {
	let year = buffer.readUInt16BE(offset);
	let month = buffer.readUInt8(offset + 2);
	let day = buffer.readUInt8(offset + 3);
	//let weekday = buffer.readUInt8(offset + 4);
	let hour = buffer.readUInt8(offset + 5);
	let min = buffer.readUInt8(offset + 6);
	let sec = buffer.readUInt8(offset + 7);
	let ns = buffer.readUInt32BE(offset + 8);

	let date;
	if (isUTC) {
		date = new Date(Date.UTC(year, month - 1,
			day, hour, min, sec, ns / 1e6))
	} else {
		date = new Date(year, month - 1,
			day, hour, min, sec, ns / 1e6);
	}

	return date;
}


```





[<-- До опису бібліотеки](README.md) 





