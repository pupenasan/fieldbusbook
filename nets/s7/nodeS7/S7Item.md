[<-- До опису бібліотеки](README.md) 

## S7Item

```js
function S7Item() { // Object
	// Зберігає оригінальну адресу
	this.addr = undefined;
	this.useraddr = undefined;

	// Перша група — властивості, пов’язані з S7 — лише вони означують адресу. 
	this.addrtype = undefined;
	this.datatype = undefined;
	this.dbNumber = undefined;
	this.bitOffset = undefined;
	this.offset = undefined;
	this.arrayLength = undefined;

	// Ці наступні властивості можуть бути обчислені з наведених вище властивостей 
    // і можуть бути перетворені на функції.
	this.dtypelen = undefined;
	this.areaS7Code = undefined;
	this.byteLength = undefined;
	this.byteLengthWithFill = undefined;

	// Зверніть увагу, що транспортні коди читання та транспортні коди запису 
    // будуть однаковими, за винятком бітів, які читаються як байти, 
    // але записуються як біти 
	this.readTransportCode = undefined;
	this.writeTransportCode = undefined;

	// Саме сюди можуть надходити дані, які надходять у пакеті, 
    // перед обчисленням значення. 
	this.byteBuffer = Buffer.alloc(8192);
	this.writeBuffer = Buffer.alloc(8192);

	// Ми використовуємо "quality buffer", щоб відстежувати, чи були запити успішними.
	// В іншому випадку дуже легко втратити слід масивів, 
    // які можуть бути лише частково повними. 
	this.qualityBuffer = Buffer.alloc(8192);
	this.writeQualityBuffer = Buffer.alloc(8192);

	// Далі у нас є властивості елементів
	this.value = undefined;
	this.writeValue = undefined;
	this.valid = false;
	this.errCode = undefined;

	// Далі ми маємо властивості результату
	this.part = undefined;
	this.maxPart = undefined;

	// Block properties
	this.isOptimized = false;
	this.resultReference = undefined;
	this.itemReference = undefined;

	// And functions...
	this.clone = function() {
		var newObj = new S7Item();
		for (var i in this) {
			if (i == 'clone') continue;
			newObj[i] = this[i];
		} return newObj;
	};

	this.badValue = function() {
		switch (this.datatype) {
			case "DT":
			case "DTZ":
			case "DTL":
			case "DTLZ":
				return new Date(NaN);
			case "REAL":
			case "LREAL":
				return 0.0;
			case "DWORD":
			case "DINT":
			case "INT":
			case "LINT":
			case "WORD":
			case "B":
			case "BYTE":
			case "TIMER":
			case "COUNTER":
				return 0;
			case "X":
				return false;
			case "C":
			case "CHAR":
			case "S":
			case "STRING":
				// Convert to string.
				return "";
			default:
				outputLog("Невідомий тип даних під час визначення неправильного значення" 
                          + " - ніколи не повинно статися. Треба було зловити раніше." +
                          this.datatype);
				return 0;
		}
	};
}
```





[<-- До опису бібліотеки](README.md) 





