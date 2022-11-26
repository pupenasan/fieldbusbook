[<-- До опису бібліотеки](README.md) 

## S7AddrToBuffer

```js
function S7AddrToBuffer(addrinfo, isWriting) {
	var thisBitOffset = 0, theReq = Buffer.from([0x12, 0x0a, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]);

	// First 3 bytes (0,1,2) is constant, sniffed from other traffic, for S7 head.
	// Next one is "byte length" - we always request X number of bytes - even for a REAL with length of 1 we read BYTES length of 4.
	theReq[3] = 0x02;  // Byte length

	// Next we write the number of bytes we are going to read.
	if (addrinfo.datatype === 'X') {
		theReq.writeUInt16BE(addrinfo.byteLength, 4);
		if (isWriting && addrinfo.arrayLength === 1) {
			// Byte length will be 1 already so no need to special case this.
			theReq[3] = 0x01;  // 1 = "BIT" length
			// We need to specify the bit offset in this case only.  Normally, when reading, we read the whole byte anyway and shift bits around.  Can't do this when writing only one bit.
			thisBitOffset = addrinfo.bitOffset;
		}
	} else if (addrinfo.datatype === 'TIMER' || addrinfo.datatype === 'COUNTER') {
		theReq.writeUInt16BE(1, 4);
		theReq.writeUInt8(addrinfo.areaS7Code, 3);
	} else {
		theReq.writeUInt16BE(addrinfo.byteLength, 4);
	}

	// Then we write the data block number.
	theReq.writeUInt16BE(addrinfo.dbNumber, 6);

	// Write our area crossing pointer.  When reading, write a bit offset of 0 - we shift the bit offset out later only when reading.
	theReq.writeUInt32BE(addrinfo.offset * 8 + thisBitOffset, 8);

	// Now we have to BITWISE OR the area code over the area crossing pointer.
	// This must be done AFTER writing the area crossing pointer as there is overlap, but this will only be noticed on large DB.
	theReq[8] |= addrinfo.areaS7Code;

	return theReq;
}
```





[<-- До опису бібліотеки](README.md) 





