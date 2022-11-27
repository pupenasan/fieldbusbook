[<-- До опису бібліотеки](README.md) 

# checkRFCData

Перевіряє тип отриманого пакету по даним  `data`, повертає:

- "fastACK" - для старих ПЛК
- `data` - дані
- обрізана `data` на довжину `FastAcknowledge`
- "error" - помилка

```js
function checkRFCData(data){
   var ret=null;
   var RFC_Version = data[0];
   var TPKT_Length = data.readInt16BE(2);
   var TPDU_Code = data[5]; //Data==0xF0 !!
   //empty fragmented frame => 0=not the last package; 1=last package 
   var LastDataUnit = data[6];

   if(RFC_Version !==0x03 && TPDU_Code !== 0xf0){
      //Перевіряє чи це пакет RFC і пакет даних
      return 'error';
   } else if ((LastDataUnit >> 7) === 0 && 
              TPKT_Length == data.length &&  
              data.length === 7){
      // Перевіряє, чи це пакет Fast Acknowledge від старіших ПЛК або WinAC, 
      // чи дані занадто довгі... 
      // Наприклад: <Buffer 03 00 00 07 02 f0 00> => data.length==7
      ret='fastACK';
   } else if ((LastDataUnit >> 7) == 1 && TPKT_Length <= data.length){
      // Перевіряє, чи це пакет FastAcknowledge  + S7Data package
      // <Buffer 03 00 00 1b 02 f0 80 32 03 00 00 00 00 00 08 00 00 00 00 
      // f0 00 00 01 00 01 00 f0> => data.length==7+20=27
      ret=data;
   } else if ((LastDataUnit >> 7) == 0  && TPKT_Length !== data.length){
      // Перевіряє, чи це пакет FastAcknowledge + 
      // FastAcknowledge package + S7Data package
      // Можливо, через те, що NodeS7 або Application занадто повільні на даний момент!
      // <Buffer 03 00 00 07 02 f0 00 03 00 00 1b 02 f0 80 32 03 00 00 00 00 
      // 00 08 00 00 00 00 f0 00 00 01 00 01 00 f0>  => data.length==7+7+20=34
      // Відріжте перший пакет швидкого підтвердження 
      ret = data.slice(7, data.length) 
   } else {
      ret='error';
   }
   return ret;
}
```



[<-- До опису бібліотеки](README.md) 





