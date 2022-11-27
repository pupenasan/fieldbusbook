[<-- До опису бібліотеки](README.md) 

# onResponse

Перевіряє отриманий пакет на коректність і відправляє його далі на обробку 

```js
NodeS7.prototype.onResponse = function(theData) {
	var self = this;
	// Перевірка дійсності пакета. Зауважте, що це пройде навіть у разі 
    // отримання від сервера відповіді "not available". 
	// Для розрахунку та перевірки довжини: 
	// data[4] = COTP header length. Normally 2.  
    // Це не включає байт довжини, тому додайте 1.
	// read(13) is parameter length.  Normally 4.
	// read(14) is data length.  (Includes item headers)
	// 12 is length of "S7 header"
	// Then we need to add 4 for TPKT header.

	// Зменште наші паралельні завдання зараз

   // НЕ ТАК ШВИДКО - тут не можна. Якщо ми закінчимо час очікування, 
   // а потім отримаємо відповідь пізніше, ми не можемо зменшити це двічі. 
   // Або CPU нас не любить. Зробіть це, якщо не rcvd.self.parallelJobsNow--;
    
	// hotfix for #78, prevents RangeErrors for undersized packets
	if (!(theData && theData.length > 6)) {
		outputLog('INVALID READ RESPONSE - DISCONNECTING');
		outputLog("The incoming packet doesn't have the required" + 
                  "minimum length of 7 bytes");
		outputLog(theData);
		self.connectionReset();
		return;
	}

	var data=checkRFCData(theData);

	if(data==="fastACK"){
		//read again and wait for the requested data
		outputLog('Fast Acknowledge received.', 0, self.connectionID);
		self.isoclient.removeAllListeners('error');
		self.isoclient.removeAllListeners('data');
		self.isoclient.on('data', function() {
			self.onResponse.apply(self, arguments);
		});
		self.isoclient.on('error', function() {
			self.readWriteError.apply(self, arguments);
		});
	} else if (data[7] === 0x32 ){//check the validy of FA+S7 package
		if (data.length > 8 && data[8] != 3) {
			outputLog('PDU type (byte 8) was returned as ' + data[8] + 
                      ' where the response PDU of 3 was expected.');
			outputLog('Maybe you are requesting more than" + 
                      "240 bytes of data in a packet?');
			outputLog(data);
			self.connectionReset();
			return null;
		}
		// Найменший прочитаний пакет пройде перевірку довжини 25. 
        // Для відповіді на запис із 1 елемента без даних довжина становитиме 22. 
		if (data.length > data.readInt16BE(2)) {
			outputLog("Було виявлено завеликий пакет. Зайва довжина є " +
                      (data.length - data.readInt16BE(2)) + ".  ");
			outputLog ("Ми припускаємо, що це тому, що було два пакети" + 
                       "надіслані ПЛК майже одночасно.");
			outputLog("Ми нарізаємо буфер і плануємо другу половину" +
                      "для подальшої обробки наступного циклу");
			// Це повторно запускає ту саму функцію з нарізаним буфером. 
            setTimeout(function() {
				self.onResponse.apply(self, arguments);
			}, 0, data.slice(data.readInt16BE(2)));  
		}

		if (data.length < data.readInt16BE(2) 
            || data.readInt16BE(2) < 22 
            || data[5] !== 0xf0 
            || data[4] + 1 + 12 + 4 
            + data.readInt16BE(13) + data.readInt16BE(15) !== data.readInt16BE(2) 
            || !(data[6] >> 7) 
            || (data[7] !== 0x32) 
            || (data[8] !== 3)) {
			outputLog('INVALID READ RESPONSE - DISCONNECTING');
			outputLog('TPKT Length From Header is ' + data.readInt16BE(2) 
                      + ' and RCV buffer length is ' + data.length 
                      + ' and COTP length is ' + data.readUInt8(4) 
                      + ' and data[6] is ' + data[6]);
			outputLog(data);
			self.connectionReset();
			return null;
		}

		//**********************   GO ON  *************************
		// Log the receive
		outputLog('Received ' + data.readUInt16BE(15) + 
                  ' bytes of S7-data from PLC.  Sequence number is ' +
                  data.readUInt16BE(11), 1, self.connectionID);

		// Перевірте порядковий номер
		var foundSeqNum; // self.readPacketArray.length - 1;
		var isReadResponse, isWriteResponse;
        
		foundSeqNum = self.findReadIndexOfSeqNum(data.readUInt16BE(11));
		if (foundSeqNum === undefined) {
			foundSeqNum = self.findWriteIndexOfSeqNum(data.readUInt16BE(11));
			if (foundSeqNum !== undefined) {
				self.writeResponse(data, foundSeqNum);
				isWriteResponse = true;
			}
		} else {
			isReadResponse = true;
			self.readResponse(data, foundSeqNum);
		}

		if ((!isReadResponse) && (!isWriteResponse)) {
			outputLog("Порядковий номер, який надійшов," + 
                      " також не був письмовою відповіддю - скидання ");
			outputLog(data);
			return null;
		}

	}else{
		outputLog('INVALID READ RESPONSE - DISCONNECTING');
		outputLog('TPKT Length From Header is ' + theData.readInt16BE(2) 
                  + ' and RCV buffer length is ' + theData.length 
                  + ' and COTP length is ' + theData.readUInt8(4) 
                  + ' and data[6] is ' + theData[6]);
		outputLog(theData);
		self.connectionReset();
		return null;
	}

}
```





[<-- До опису бібліотеки](README.md) 





