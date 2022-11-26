[<-- До опису бібліотеки](README.md) 

```js
function writeDTL(date, buffer, offset, isUTC) {
	if (!(date instanceof Date)) {
		if (date >= 0 && date < 9223382836854) {
			// is between "1970-01-01T00:00:00.000Z" and "2262-04-11T23:47:16.854Z" in JS epoch
			// as per data type's range definition
			date = new Date(date);
		} else {
			outputLog("Unsupported value of [" + date + "] for writing data of type DATE_AND_TIME. Skipping item");
			return;
		}
	}

	if (isUTC) {
		buffer.writeUInt16BE(date.getUTCFullYear(), offset);
		buffer.writeUInt8(date.getUTCMonth() + 1, offset + 2);
		buffer.writeUInt8(date.getUTCDate(), offset + 3);
		buffer.writeUInt8(date.getUTCDay() + 1, offset + 4);
		buffer.writeUInt8(date.getUTCHours(), offset + 5);
		buffer.writeUInt8(date.getUTCMinutes(), offset + 6);
		buffer.writeUInt8(date.getUTCSeconds(), offset + 7);
		buffer.writeUInt32BE(date.getUTCMilliseconds() * 1e6, offset + 8);
	} else {
		buffer.writeUInt16BE(date.getFullYear(), offset);
		buffer.writeUInt8(date.getMonth() + 1, offset + 2);
		buffer.writeUInt8(date.getDate(), offset + 3);
		buffer.writeUInt8(date.getDay() + 1, offset + 4);
		buffer.writeUInt8(date.getHours(), offset + 5);
		buffer.writeUInt8(date.getMinutes(), offset + 6);
		buffer.writeUInt8(date.getSeconds(), offset + 7);
		buffer.writeUInt32BE(date.getMilliseconds() * 1e6, offset + 8);
	}
}
```





[<-- До опису бібліотеки](README.md) 





