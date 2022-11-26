[<-- До опису бібліотеки](README.md) 

```js
function writeDT(date, buffer, offset, isUTC){
	if (!(date instanceof Date)) {
		if (date > 631152000000 && date < 3786911999999) {
			// is between "1990-01-01T00:00:00.000Z" and "2089-12-31T23:59:59.999Z" in JS epoch
			// as per data type's range definition
			date = new Date(date);
		} else {
			outputLog("Unsupported value of [" + date + "] for writing data of type DATE_AND_TIME. Skipping item");
			return;
		}
	}

	if (isUTC) {
		buffer.writeUInt8(toBCD(date.getUTCFullYear() % 100), offset);
		buffer.writeUInt8(toBCD(date.getUTCMonth() + 1), offset + 1);
		buffer.writeUInt8(toBCD(date.getUTCDate()), offset + 2);
		buffer.writeUInt8(toBCD(date.getUTCHours()), offset + 3);
		buffer.writeUInt8(toBCD(date.getUTCMinutes()), offset + 4);
		buffer.writeUInt8(toBCD(date.getUTCSeconds()), offset + 5);
		buffer.writeUInt8(toBCD((date.getUTCMilliseconds() / 10) >> 0), offset + 6);
		buffer.writeUInt8(toBCD(((date.getUTCMilliseconds() % 10) * 10) + (date.getUTCDay() + 1)), offset + 7);
	} else {
		buffer.writeUInt8(toBCD(date.getFullYear() % 100), offset);
		buffer.writeUInt8(toBCD(date.getMonth() + 1), offset + 1);
		buffer.writeUInt8(toBCD(date.getDate()), offset + 2);
		buffer.writeUInt8(toBCD(date.getHours()), offset + 3);
		buffer.writeUInt8(toBCD(date.getMinutes()), offset + 4);
		buffer.writeUInt8(toBCD(date.getSeconds()), offset + 5);
		buffer.writeUInt8(toBCD((date.getMilliseconds() / 10) >> 0), offset + 6);
		buffer.writeUInt8(toBCD(((date.getMilliseconds() % 10) * 10) + (date.getDay() + 1)), offset + 7);
	}
}
```





[<-- До опису бібліотеки](README.md) 





