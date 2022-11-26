# Тестовий клієнт

```js
var nodes7 = require('nodes7'); // Це назва пакета, якщо репозиторій клоновано, вам може знадобитися вимагати «nodeS7» із S у верхньому регістрі
var conn = new nodes7;
var doneReading = false;
var doneWriting = false;

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

conn.initiateConnection({ port: 102, host: '192.168.56.102', rack: 0, slot: 2, debug: true }, connected); // slot 2 for 300/400, slot 1 for 1200/1500, change debug to true to get more info
// conn.initiateConnection({port: 102, host: '192.168.0.2', localTSAP: 0x0100, remoteTSAP: 0x0200, timeout: 8000, doNotOptimize: true}, connected);
// локальний і віддалений TSAP також можна вказати безпосередньо. Опція тайм-ауту вказує час очікування TCP.

function connected(err) {
  if (typeof(err) !== "undefined") {
    // У нас сталася помилка. Можливо, ПЛК недоступний.
    console.log(err);
    process.exit();
  }
  conn.setTranslationCB(function(tag) { return variables[tag]; }); // Це встановлює "translation", щоб ми могли працювати з назвами об’єктів 
  conn.addItems(['TEST1', 'TEST4']);
  conn.addItems('TEST6');
  // conn.removeItems(['TEST2', 'TEST3']); // We could do this.
  // conn.writeItems(['TEST5', 'TEST6'], [ 867.5309, 9 ], valuesWritten); // Ви також можете написати масив елементів.
  // conn.writeItems('TEST7', [666, 777], valuesWritten); // Ви також можете написати один елемент масиву.
  conn.writeItems('TEST3', true, valuesWritten); // Це записує один логічний елемент (один біт) у значення true
  conn.readAllItems(valuesReady);
}

function valuesReady(anythingBad, values) {
  if (anythingBad) { console.log("SOMETHING WENT WRONG READING VALUES!!!!"); }
  console.log(values);
  doneReading = true;
  if (doneWriting) { process.exit(); }
}

function valuesWritten(anythingBad) {
  if (anythingBad) { console.log("SOMETHING WENT WRONG WRITING VALUES!!!!"); }
  console.log("Done writing.");
  doneWriting = true;
  if (doneReading) { process.exit(); }
}
```

