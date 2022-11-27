[<-- До опису бібліотеки](README.md) 

# stringToS7Addr

Повертає обєкт `S7Item` за текстовим представленням

```js
stringToS7Addr (addr, useraddr, cParam)
```

`addr` - адреса в форматі S7

`useraddr` - користувацька назва змінної

- `'_COMMERR'` - ця змінна повертає значення true, коли виникає помилка зв’язку

`cParam` - потрібно для визначення представлення даних `cParam.wdtAsUTC`

```js
stringToS7Addr('MR4', `TEST1`, self.connectionParams)
```

Приклади `useraddr:addr` :

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
```



```js
function stringToS7Addr(addr, useraddr, cParam) {
	"use strict";
	var theItem, splitString, splitString2;
	// Спеціальний випадок для статусу помилки зв’язку – ця змінна 
    // повертає значення true, коли виникає помилка зв’язку
	if (useraddr === '_COMMERR') { return undefined; } 

	theItem = new S7Item();
	splitString = addr.split(',');
	if (splitString.length === 0 || splitString.length > 2) {
		outputLog("Помилка - Рядок не вдалося розділити належним чином.");
		return undefined;
	}

	if (splitString.length > 1) { // Має бути типу DB, напиклад 'DB1,REAL8'
		theItem.addrtype = 'DB';  // Hard code
		splitString2 = splitString[1].split('.');
        // Убираємо числа
		theItem.datatype = splitString2[0].replace(/[0-9]/gi, '').toUpperCase(); 
		if (theItem.datatype === 'X' && splitString2.length === 3) {
			theItem.arrayLength = parseInt(splitString2[2], 10);
		} else if ((theItem.datatype === 'S' || theItem.datatype === 'STRING') 
                   && splitString2.length === 3) {
            // З рядками додайте 2 до довжини через заголовок S7
			theItem.dtypelen = parseInt(splitString2[1], 10) + 2;
            // Для рядків довжина масиву тепер дорівнює кількості рядків
			theItem.arrayLength = parseInt(splitString2[2], 10);  
		} else if ((theItem.datatype === 'S' || theItem.datatype === 'STRING') 
                   && splitString2.length === 2) {
            // З рядками додайте 2 до довжини через заголовок S7
			theItem.dtypelen = parseInt(splitString2[1], 10) + 2;
			theItem.arrayLength = 1;
		} else if (theItem.datatype !== 'X' && splitString2.length === 2) {
			theItem.arrayLength = parseInt(splitString2[1], 10);
		} else {
			theItem.arrayLength = 1;
		}
		if (theItem.arrayLength <= 0) {
			outputLog('Zero length arrays not allowed, returning undefined');
			return undefined;
		}

		// Отримайте номер блоку даних з першої частини.
		theItem.dbNumber = parseInt(splitString[0].replace(/[A-z]/gi, ''), 10);

		// Отримайте байтове зміщення блоку даних від другої частини, виключаючи символи.
        // Зауважте, що на цьому етапі ми можемо пропустити деяку інформацію, 
        // як-от літеру "T" у кінці, яка вказує на тип даних TIME, 
        // тип даних DATE або тип даних DT. Ми їх ігноруємо.
		// Це в списку TODO.
        // Позбутися символів
		theItem.offset = parseInt(splitString2[0].replace(/[A-z]/gi, ''), 10);  

		// Отримати бітове зміщення
		if (splitString2.length > 1 && theItem.datatype === 'X') {
			theItem.bitOffset = parseInt(splitString2[1], 10);
			if (theItem.bitOffset > 7) {
				outputLog("Invalid bit offset specified for address " + addr);
				return undefined;
			}
		}
	} else { // Не має бути DB. Ми знаємо, що коми немає. 
		splitString2 = addr.split('.');

		switch (splitString2[0].replace(/[0-9]/gi, '')) {
			/* We do have the memory areas:
			  "input", "peripheral input", "output", 
			  "peripheral output", ",marker", "counter" 
			  and "timer" as I, PI, Q, PQ, M, C and T.
			   Datablocks are handles somewere else.
			   We do have the data types:
			   "bit", "byte", "char", "word", "int16", "dword", 
			   "int32", "real" as X, B, C, W, I, DW, DI and R
			   What about "uint16", "uint32"
			*/

		/* Усі стилі периферійних операцій вводу-виводу (доступ до бітів заборонений) */
			case "PIB":
			case "PEB":
			case "PQB":
			case "PAB":
				theItem.addrtype = "P";
				theItem.datatype = "BYTE";
				break;
			case "PIC":
			case "PEC":
			case "PQC":
			case "PAC":
				theItem.addrtype = "P";
				theItem.datatype = "CHAR";
				break;
			case "PIW":
			case "PEW":
			case "PQW":
			case "PAW":
				theItem.addrtype = "P";
				theItem.datatype = "WORD";
				break;
			case "PII":
			case "PEI":
			case "PQI":
			case "PAI":
				theItem.addrtype = "P";
				theItem.datatype = "INT";
				break;
			case "PID":
			case "PED":
			case "PQD":
			case "PAD":
				theItem.addrtype = "P";
				theItem.datatype = "DWORD";
				break;
			case "PIDI":
			case "PEDI":
			case "PQDI":
			case "PADI":
				theItem.addrtype = "P";
				theItem.datatype = "DINT";
				break;
			case "PIR":
			case "PER":
			case "PQR":
			case "PAR":
				theItem.addrtype = "P";
				theItem.datatype = "REAL";
				break;

/* Усі стилі стандартних входів (на відміну від периферійних входів)  */
			case "I":
			case "E":
				theItem.addrtype = "I";
				theItem.datatype = "X";
				break;
			case "IB":
			case "EB":
				theItem.addrtype = "I";
				theItem.datatype = "BYTE";
				break;
			case "IC":
			case "EC":
				theItem.addrtype = "I";
				theItem.datatype = "CHAR";
				break;
			case "IW":
			case "EW":
				theItem.addrtype = "I";
				theItem.datatype = "WORD";
				break;
			case "II":
			case "EI":
				theItem.addrtype = "I";
				theItem.datatype = "INT";
				break;
			case "ID":
			case "ED":
				theItem.addrtype = "I";
				theItem.datatype = "DWORD";
				break;
			case "IDI":
			case "EDI":
				theItem.addrtype = "I";
				theItem.datatype = "DINT";
				break;
			case "IR":
			case "ER":
				theItem.addrtype = "I";
				theItem.datatype = "REAL";
				break;
			case "ILR":
			case "ELR":
				theItem.addrtype = "I";
				theItem.datatype = "LREAL";
				break;
			case "ILI":
			case "ELI":
				theItem.addrtype = "I";
				theItem.datatype = "LINT";
				break;
/* All styles of standard outputs (in oposit to peripheral outputs) */
			case "Q":
			case "A":
				theItem.addrtype = "Q";
				theItem.datatype = "X";
				break;
			case "QB":
			case "AB":
				theItem.addrtype = "Q";
				theItem.datatype = "BYTE";
				break;
			case "QC":
			case "AC":
				theItem.addrtype = "Q";
				theItem.datatype = "CHAR";
				break;
			case "QW":
			case "AW":
				theItem.addrtype = "Q";
				theItem.datatype = "WORD";
				break;
			case "QI":
			case "AI":
				theItem.addrtype = "Q";
				theItem.datatype = "INT";
				break;
			case "QD":
			case "AD":
				theItem.addrtype = "Q";
				theItem.datatype = "DWORD";
				break;
			case "QDI":
			case "ADI":
				theItem.addrtype = "Q";
				theItem.datatype = "DINT";
				break;
			case "QR":
			case "AR":
				theItem.addrtype = "Q";
				theItem.datatype = "REAL";
				break;
			case "QLR":
			case "ALR":
				theItem.addrtype = "Q";
				theItem.datatype = "LREAL";
				break;
			case "QLI":
			case "ALI":
				theItem.addrtype = "Q";
				theItem.datatype = "LINT";
				break;
/* Всі стилі маркера */
			case "M":
				theItem.addrtype = "M";
				theItem.datatype = "X";
				break;
			case "MB":
				theItem.addrtype = "M";
				theItem.datatype = "BYTE";
				break;
			case "MC":
				theItem.addrtype = "M";
				theItem.datatype = "CHAR";
				break;
			case "MW":
				theItem.addrtype = "M";
				theItem.datatype = "WORD";
				break;
			case "MI":
				theItem.addrtype = "M";
				theItem.datatype = "INT";
				break;
			case "MD":
				theItem.addrtype = "M";
				theItem.datatype = "DWORD";
				break;
			case "MDI":
				theItem.addrtype = "M";
				theItem.datatype = "DINT";
				break;
			case "MR":
				theItem.addrtype = "M";
				theItem.datatype = "REAL";
				break;
			case "MLR":
				theItem.addrtype = "M";
				theItem.datatype = "LREAL";
				break;
			case "MLI":
				theItem.addrtype = "M";
				theItem.datatype = "LINT";
				break;
/* Timer */
			case "T":
				theItem.addrtype = "T";
				theItem.datatype = "TIMER";
				break;

/* Counter */
			case "C":
				theItem.addrtype = "C";
				theItem.datatype = "COUNTER";
				break;

			default:
				outputLog('Failed to find a match for ' + splitString2[0]);
				return undefined;
		}

		theItem.bitOffset = 0;
		if (splitString2.length > 1 && theItem.datatype === 'X') { // Bit and bit array
			theItem.bitOffset = parseInt(splitString2[1].replace(/[A-z]/gi, ''), 10);
			if (splitString2.length > 2) {  // Bit array only
				theItem.arrayLength = parseInt(splitString2[2].replace(/[A-z]/gi, '')
                                               , 10);
			} else {
				theItem.arrayLength = 1;
			}
		} // Bit and bit array
        else if (splitString2.length > 1 && theItem.datatype !== 'X') { 
			theItem.arrayLength = parseInt(splitString2[1].replace(/[A-z]/gi, ''), 10);
		} else {
			theItem.arrayLength = 1;
		}
		theItem.dbNumber = 0;
		theItem.offset = parseInt(splitString2[0].replace(/[A-z]/gi, ''), 10);
	}

	if (theItem.datatype === 'DI') {
		theItem.datatype = 'DINT';
	}
	if (theItem.datatype === 'I') {
		theItem.datatype = 'INT';
	}
	if (theItem.datatype === 'DW' || theItem.datatype === 'DWT') {
		theItem.datatype = 'DWORD';
	}
	if (theItem.datatype === 'WDT') {
		if (cParam.wdtAsUTC) {
			theItem.datatype = 'DTZ';
		} else {
			theItem.datatype = 'DT';
		}
	}
	if (theItem.datatype === 'W') {
		theItem.datatype = 'WORD';
	}
	if (theItem.datatype === 'R') {
		theItem.datatype = 'REAL';
	}
	if (theItem.datatype === 'LR') {
		theItem.datatype = 'LREAL';
	}
	if (theItem.datatype === 'LI') {
		theItem.datatype = 'LINT';
	}
	switch (theItem.datatype) {
		case "DTL":
		case "DTLZ":
			theItem.dtypelen = 12;
			break;
		case "LREAL":
		case "LINT":
		case "DT":
		case "DTZ":
			theItem.dtypelen = 8;
			break;
		case "REAL":
		case "DWORD":
		case "DINT":
			theItem.dtypelen = 4;
			break;
		case "INT":
		case "WORD":
		case "TIMER":
		case "COUNTER":
			theItem.dtypelen = 2;
			break;
		case "X":
		case "B":
		case "C":
		case "BYTE":
		case "CHAR":
			theItem.dtypelen = 1;
			break;
		case "S":
		case "STRING":
			// For strings, arrayLength and dtypelen were assigned during parsing.
			break;
		default:
			outputLog("Unknown data type " + theItem.datatype);
			return undefined;
	}

	// Default
	theItem.readTransportCode = 0x04;

	switch (theItem.addrtype) {
		case "DB":
		case "DI":
			theItem.areaS7Code = 0x84;
			break;
		case "I":
		case "E":
			theItem.areaS7Code = 0x81;
			break;
		case "Q":
		case "A":
			theItem.areaS7Code = 0x82;
			break;
		case "M":
			theItem.areaS7Code = 0x83;
			break;
		case "P":
			theItem.areaS7Code = 0x80;
			break;
		case "C":
			theItem.areaS7Code = 0x1c;
			theItem.readTransportCode = 0x09;
			break;
		case "T":
			theItem.areaS7Code = 0x1d;
			theItem.readTransportCode = 0x09;
			break;
		default:
			outputLog("Unknown memory area entered - " + theItem.addrtype);
			return undefined;
	}

	if (theItem.datatype === 'X' && theItem.arrayLength === 1) {
		theItem.writeTransportCode = 0x03;
	} else {
		theItem.writeTransportCode = theItem.readTransportCode;
	}

	// Save the address from the argument for later use and reference
	theItem.addr = addr;
	if (useraddr === undefined) {
		theItem.useraddr = addr;
	} else {
		theItem.useraddr = useraddr;
	}

	if (theItem.datatype === 'X') {
		theItem.byteLength = Math.ceil((theItem.bitOffset + theItem.arrayLength) / 8);
	} else {
		theItem.byteLength = theItem.arrayLength * theItem.dtypelen;
	}

	theItem.byteLengthWithFill = theItem.byteLength;
    // S7 додасть байт-заповнювач. 
    // Використовуйте цю очікувану довжину відповіді для обчислень PDU.
	if (theItem.byteLengthWithFill % 2) { theItem.byteLengthWithFill += 1; }  

	return theItem;
}
```





[<-- До опису бібліотеки](README.md) 





