

# PDU (protocol data unit)

This is the central part of packets exchanged in S7-Communication. 

A PDU cosists of:

A 10 or 12 byte header

A parameter area

A data area

## Header:

| Position | meaning              | possible values |
| -------- | -------------------- | --------------- |
| 0        | allways 0x32         |                 |
| 1        | type                 | 1,2,3 or 7      |
| 2,3      | unknown              | 0               |
| 4,5      | sequence number      |                 |
| 6,7      | length of parameters |                 |
| 8,9      | length of data       |                 |
| 10,11    | error code           |                 |

## Parameters:

| Position | meaning                    | possible values |
| -------- | -------------------------- | --------------- |
| 0        | a function number          |                 |
| rest     | depends on function number |                 |

## Data:

| Position | meaning                    | possible values |
| -------- | -------------------------- | --------------- |
| rest     | depends on function number |                 |

## Parameters for read request:

| Position | meaning                     | possible values |
| -------- | --------------------------- | --------------- |
| 0        | function number for read    | 4               |
| 1        | number of items to read     | 1..20           |
| 2..      | item adresses, 12 byte each |                 |

### Forming the item address:

| Position | meaning                     | possible values                          |
| -------- | --------------------------- | ---------------------------------------- |
| 0,1,2    | unknown                     | allways 0x12, 0x0a, 0x10                 |
| 3        | transport size or unit size | 1=single bit, 2=byte, 4=word             |
| 4,5      | length in byte              |                                          |
| 6,7      | number of data block        | 0 for ares other than data block         |
| 8        | area code                   | see area                                 |
| 9,10,11  | Start address in bits.      | multiples of 8, if unit size is not bits |

## read response:

| Position | meaning                                | possible values |
| -------- | -------------------------------------- | --------------- |
| 0        | function number for read               | 4               |
| 1        | number read items                      | 1..20           |
| 2..      | items, 4 byte "data header" +data each |                 |

### Data header:

| Position | meaning                     | possible values                                              |
| -------- | --------------------------- | ------------------------------------------------------------ |
| 0        | return code                 | 0xFF means ok, data follows after this header. Other codes give reasons why no data is returned. |
| 1        | transport size or unit size | 4=single bit, 9=byte                                         |
| 2,3      | length in bits              |                                                              |

# Error codes

## Error codes reported in 12 byte headers of type 2 and 3 PDUs

| 0      | ok                                       |
| ------ | ---------------------------------------- |
| 0x8000 | function already occupied.               |
| 0x8001 | not allowed in current operating status. |
| 0x8101 | hardware fault                           |
| 0x8103 | object access not allowed.               |
| 0x8104 | context is not supported.                |
| 0x8105 | invalid address.                         |
| 0x8106 | data type not supported.                 |
| 0x8107 | data type not consistent.                |
| 0x810A | object does not exist.                   |
| 0x8500 | incorrect PDU size.                      |
| 0x8702 | address invalid."                        |
| 0xd201 | block name syntax error.                 |
| 0xd202 | syntax error function parameter.         |
| 0xd203 | syntax error block type.                 |
| 0xd204 | no linked block in storage medium.       |
| 0xd205 | object already exists.                   |
| 0xd206 | object already exists.                   |
| 0xd207 | block exists in EPROM.                   |
| 0xd209 | block does not exist.                    |
| 0xd20e | no block does not exist.                 |
| 0xd210 | block number too big.                    |

## Константи

nodave.h

### some frequently used ASCII control codes:

```c
#define DLE 0x10
#define ETX 0x03
#define STX 0x02
#define SYN 0x16
#define NAK 0x15
#define EOT 0x04    //  for S5
#define ACK 0x06    //  for S5
```



### Protocol types to be used with new Interface:

```c
#define daveProtoMPI	0	/* MPI for S7 300/400 */
#define daveProtoMPI2	1	/* MPI for S7 300/400, "Andrew's version" without STX */
#define daveProtoMPI3	2	/* MPI for S7 300/400, Step 7 Version, not yet implemented */
#define daveProtoMPI4	3	/* MPI for S7 300/400, "Andrew's version" with STX */

#define daveProtoPPI	10	/* PPI for S7 200 */

#define daveProtoAS511	20	/* S5 programming port protocol */

#define daveProtoS7online 50	/* use s7onlinx.dll for transport */

#define daveProtoISOTCP	122	/* ISO over TCP */
#define daveProtoISOTCP243 123	/* ISO over TCP with CP243 */

#define daveProtoMPI_IBH 223	/* MPI with IBH NetLink MPI to ethernet gateway */
#define daveProtoPPI_IBH 224	/* PPI with IBH NetLink PPI to ethernet gateway */

#define daveProtoNLpro 230	/* MPI with NetLink Pro MPI to ethernet gateway */

#define daveProtoUserTransport 255	/* Libnodave will pass the PDUs of S7 Communication to user */
					/* defined call back functions. */
```

### Communication types

```c
#define davePGCommunication 1	/* communication with programming device (PG) (default in Libnodave) */
#define daveProgrammerCommunication 1	/* communication with programming device (PG) (default in Libnodave) */
#define daveOPCommunication 2	/* communication with operator panel (OP)) */
#define daveS7BasicCommunication 3	/* communication with another CPU ? */
```

###  Some S7 communication function codes (yet unused ones may be incorrect)

```c
#define daveFuncOpenS7Connection	0xF0
#define daveFuncRead			0x04
#define daveFuncWrite			0x05
#define daveFuncRequestDownload		0x1A
#define daveFuncDownloadBlock		0x1B
#define daveFuncDownloadEnded		0x1C
#define daveFuncStartUpload		0x1D
#define daveFuncUpload			0x1E
#define daveFuncEndUpload		0x1F
#define daveFuncInsertBlock		0x28
```

### S7 specific constants

```c
#define daveBlockType_OB  '8'
#define daveBlockType_DB  'A'
#define daveBlockType_SDB 'B'
#define daveBlockType_FC  'C'
#define daveBlockType_SFC 'D'
#define daveBlockType_FB  'E'
#define daveBlockType_SFB 'F'

#define daveS5BlockType_DB  0x01
#define daveS5BlockType_SB  0x02
#define daveS5BlockType_PB  0x04
#define daveS5BlockType_FX  0x05
#define daveS5BlockType_FB  0x08
#define daveS5BlockType_DX  0x0C
#define daveS5BlockType_OB  0x10
```

### Use these constants for parameter "area" in daveReadBytes and daveWriteBytes

```c
#define daveSysInfo 0x3		/* System info of 200 family */
#define daveSysFlags  0x5	/* System flags of 200 family */
#define daveAnaIn  0x6		/* analog inputs of 200 family */
#define daveAnaOut  0x7		/* analog outputs of 200 family */

#define daveP 0x80    		/* direct peripheral access */
#define daveInputs 0x81    
#define daveOutputs 0x82    
#define daveFlags 0x83
#define daveDB 0x84	/* data blocks */
#define daveDI 0x85	/* instance data blocks */
#define daveLocal 0x86 	/* not tested */
#define daveV 0x87	/* don't know what it is */
#define daveCounter 28	/* S7 counters */
#define daveTimer 29	/* S7 timers */
#define daveCounter200 30	/* IEC counters (200 family) */
#define daveTimer200 31		/* IEC timers (200 family) */
#define daveSysDataS5 0x86	/* system data area ? */
#define daveRawMemoryS5 0		/* just the raw memory */
```

### Result codes. 

Genarally, 0 means ok,

  \>0 are results (also errors) reported by the PLC

  <0 means error reported by library code.

```c
#define daveResOK 0				/* means all ok */
#define daveResNoPeripheralAtAddress 1		/* CPU tells there is no peripheral at address */
#define daveResMultipleBitsNotSupported 6 	/* CPU tells it does not support to read a bit block with a */
						/* length other than 1 bit. */
#define daveResItemNotAvailable200 3		/* means a a piece of data is not available in the CPU, e.g. */
						/* when trying to read a non existing DB or bit bloc of length<>1 */
						/* This code seems to be specific to 200 family. */
					    
#define daveResItemNotAvailable 10		/* means a a piece of data is not available in the CPU, e.g. */
						/* when trying to read a non existing DB */

#define daveAddressOutOfRange 5			/* means the data address is beyond the CPUs address range */
#define daveWriteDataSizeMismatch 7		/* means the write data size doesn't fit item size */
#define daveResCannotEvaluatePDU -123    	/* PDU is not understood by libnodave */
#define daveResCPUNoData -124 
#define daveUnknownError -125 
#define daveEmptyResultError -126 
#define daveEmptyResultSetError -127 
#define daveResUnexpectedFunc -128 
#define daveResUnknownDataUnitSize -129
#define daveResNoBuffer -130
#define daveNotAvailableInS5 -131
#define daveResInvalidLength -132
#define daveResInvalidParam -133
#define daveResNotYetImplemented -134

#define daveResShortPacket -1024 
#define daveResTimeout -1025 
```

### Some definitions for debugging:

```c
#define daveDebugRawRead  	0x01	/* Show the single bytes received */
#define daveDebugSpecialChars  	0x02	/* Show when special chars are read */
#define daveDebugRawWrite	0x04	/* Show the single bytes written */
#define daveDebugListReachables 0x08	/* Show the steps when determine devices in MPI net */
#define daveDebugInitAdapter 	0x10	/* Show the steps when Initilizing the MPI adapter */
#define daveDebugConnect 	0x20	/* Show the steps when connecting a PLC */
#define daveDebugPacket 	0x40
#define daveDebugByte 		0x80
#define daveDebugCompare 	0x100
#define daveDebugExchange 	0x200
#define daveDebugPDU 		0x400	/* debug PDU handling */
#define daveDebugUpload		0x800	/* debug PDU loading program blocks from PLC */
#define daveDebugMPI 		0x1000
#define daveDebugPrintErrors	0x2000	/* Print error messages */
#define daveDebugPassive 	0x4000

#define daveDebugErrorReporting	0x8000
#define daveDebugOpen		0x10000  /* print messages in openSocket and setPort */

#define daveDebugAll 0x1ffff
```

### IBH-NetLink packet types:

```c
#define _davePtEmpty -2
#define _davePtMPIAck -3
#define _davePtUnknownMPIFunc -4
#define _davePtUnknownPDUFunc -5
#define _davePtReadResponse 1
#define _davePtWriteResponse 2
```

### Some data types

```c
#define uc unsigned char
#define us unsigned short
#define u32 unsigned int
```

### Helper struct to manage PDUs

This is NOT the part of the packet I would call PDU, but a set of pointers that ease access to the "private parts" of a PDU.

```c
typedef struct {
    uc * header;	/* pointer to start of PDU (PDU header) */
    uc * param;		/* pointer to start of parameters inside PDU */
    uc * data;		/* pointer to start of data inside PDU */
    uc * udata;		/* pointer to start of data inside PDU */
    int hlen;		/* header length */
    int plen;		/* parameter length */
    int dlen;		/* data length */
    int udlen;		/* user or result data length */
} PDU;
```

### Definitions of prototypes for the protocol specific functions.

The library "switches"   protocol by setting pointers to the protol specific implementations.

```c
typedef int (DECL2 *  _initAdapterFunc) (daveInterface * );
typedef int (DECL2 *  _connectPLCFunc) (daveConnection *);
typedef int (DECL2 * _disconnectPLCFunc) (daveConnection *);
typedef int (DECL2 * _disconnectAdapterFunc) (daveInterface * );
typedef int (DECL2 * _exchangeFunc) (daveConnection *, PDU *);
typedef int (DECL2 * _sendMessageFunc) (daveConnection *, PDU *);
typedef int (DECL2 * _getResponseFunc) (daveConnection *);
typedef int (DECL2 * _listReachablePartnersFunc) (daveInterface * di, char * buf); // changed to unsigned char because it is a copy of an uc buffer
```

Definitions of prototypes for i/O functions.

```c
typedef int (DECL2 *  _writeFunc) (daveInterface *, char *, int); // changed to char because char is what system read/write expects
typedef int (DECL2 *  _readFunc) (daveInterface *, char *, int);
```

### struct _daveInterface

This groups an interface together with some information about it's properties   in the library's context.

```c
struct _daveInterface {
    tmotype timeout;	/* Timeout in microseconds used in transort. */
    _daveOSserialType fd; /* some handle for the serial interface */
    int localMPI;	/* the adapter's MPI address */
    
    int users;		/* a counter used when multiple PLCs are accessed via */
			/* the same serial interface and adapter. */
    char * name;	/* just a name that can be used in programs dealing with multiple */
			/* daveInterfaces */
    int protocol;	/* The kind of transport protocol used on this interface. */
    int speed;		/* The MPI or Profibus speed */
    int ackPos;		/* position of some packet number that has to be repeated in ackknowledges */
    int nextConnection;
    _initAdapterFunc initAdapter;		/* pointers to the protocol */
    _connectPLCFunc connectPLC;			/* specific implementations */
    _disconnectPLCFunc disconnectPLC;		/* of these functions */
    _disconnectAdapterFunc disconnectAdapter;
    _exchangeFunc exchange;
    _sendMessageFunc sendMessage;
    _getResponseFunc getResponse;
    _listReachablePartnersFunc listReachablePartners;
    char realName[20];
    _readFunc ifread;
    _writeFunc ifwrite;
    int seqNumber;
};

EXPORTSPEC daveInterface * DECL2 daveNewInterface(_daveOSserialType nfd, char * nname, int localMPI, int protocol, int speed);
EXPORTSPEC daveInterface * DECL2 davePascalNewInterface(_daveOSserialType* nfd, char * nname, int localMPI, int protocol, int speed);
```

### IBHpacket

```c
typedef struct {
    uc ch1;	// logical connection or channel ?
    uc ch2;	// logical connection or channel ?
    uc len;	// number of bytes counted from the ninth one.
    uc packetNumber;	// a counter, response packets refer to request packets
    us sFlags;		// my guess
    us rFlags;		// my interpretation
} IBHpacket;
```

Header for MPI packets on IBH-NetLink:

```c
typedef struct {
    uc src_conn;
    uc dst_conn;
    uc MPI;
    uc localMPI;
    uc len;
    uc func;
    uc packetNumber;
} MPIheader;

typedef struct {
    uc src_conn;
    uc dst_conn;
    uc MPI;
    uc xxx1;
    uc xxx2;
    uc xx22;
    uc len;
    uc func;
    uc packetNumber;
}  MPIheader2;

typedef struct _daveS5AreaInfo  {
    int area;
    int DBnumber;
    int address;
    int len;
    struct _daveS5AreaInfo * next;
} daveS5AreaInfo;

typedef struct _daveS5cache {
    int PAE;	// start of inputs
    int PAA;	// start of outputs
    int flags;	// start of flag (marker) memory
    int timers;	// start of timer memory
    int counters;// start of counter memory
    int systemData;// start of system data
    daveS5AreaInfo * first;
} daveS5cache;


typedef struct _daveRoutingData {
	int connectionType;
	int destinationType; 	// destinationIsIP=DestinationIsIP;
	int subnetID1;
	int subnetID2;
	int subnetID3;
	int PLCadrsize;
	uc  PLCadr[4];		// currently, IP is maximum. Maybe there could be MAC adresses for Industrial Ethernet?
} daveRoutingData;
```

### struct _daveConnection

This holds data for a PLC connection;

```c
struct _daveConnection {
    int AnswLen;	/* length of last message */
    uc * resultPointer;	/* used to retrieve single values from the result byte array */
    int maxPDUlength;
    int MPIAdr;		/* The PLC's address */
    daveInterface * iface; /* pointer to used interface */
    int needAckNumber;	/* message number we need ackknowledge for */
    int PDUnumber; 	/* current PDU number */
    int ibhSrcConn;
    int ibhDstConn;
    uc msgIn[daveMaxRawLen];
    uc msgOut[daveMaxRawLen];
    uc * _resultPointer;
    int PDUstartO;	/* position of PDU in outgoing messages. This is different for different transport methodes. */
    int PDUstartI;	/* position of PDU in incoming messages. This is different for different transport methodes. */
    int rack;		/* rack number for ISO over TCP */
    int slot;		/* slot number for ISO over TCP */
    int connectionNumber;
    int connectionNumber2;
    uc 	messageNumber;  /* current MPI message number */
    uc	packetNumber;	/* packetNumber in transport layer */
    void * hook;	/* used in CPU/CP simulation: pointer to the rest we have to send if message doesn't fit in a single packet */
    daveS5cache * cache; /* used in AS511: We cache addresses of memory areas and datablocks here */
    int TPDUsize; 		// size of TPDU for ISO over TCP
    int partPos;  		// remember position for ISO over TCP fragmentation
    int routing;		// nonzero means routing enabled
    int communicationType;		// (1=PG Communication,2=OP Communication,3=Step7Basic Communication)
    daveRoutingData routingData;
}; 

EXPORTSPEC void DECL2 daveSetRoutingDestination(daveConnection * dc, int subnet1,int subnet3,int adrsize, uc* plcadr);
/* 
	    void * Destination, 
	    int DestinationIsIP, 
	    int rack, int slot, 
	    int routing, 
	    int routingSubnetFirst, 
	    int routingSubnetSecond, 
	    int routingRack, 
	    int routingSlot, 
	    void * routingDestination, 
	    int routingDestinationIsIP, 
	    int ConnectionType, 
	    int routingConnectionType) {
*/

EXPORTSPEC void DECL2 daveSetCommunicationType(daveConnection * dc,  int communicationType);
/*
EXPORTSPEC daveConnection * DECL2 daveNewEonnection(daveInterface * di, 
	    void * Destination, 
	    int DestinationIsIP, 
	    int rack, int slot, 
	    int routing, 
	    int routingSubnetFirst, 
	    int routingSubnetSecond, 
	    int routingRack, 
	    int routingSlot, 
	    void * routingDestination, 
	    int routingDestinationIsIP, 
	    int ConnectionType, 
	    int routingConnectionType) {
*/
/*
    Setup a new connection structure using an initialized
    daveInterface and PLC's MPI address.
*/
EXPORTSPEC daveConnection * DECL2 daveNewConnection(daveInterface * di, int MPI,int rack, int slot);
```



```c
typedef struct {
    uc type[2];
    unsigned short count;
} daveBlockTypeEntry;

typedef struct {
    unsigned short number;
    uc type[2];
} daveBlockEntry;

typedef struct {
    uc type[2];
    uc x1[2];  /* 00 4A */
    uc w1[2];  /* some word var? */
    char pp[2]; /* allways 'pp' */
    uc x2[4];  /* 00 4A */
    unsigned short number; /* the block's number */
    uc x3[26];  /* ? */
    unsigned short length; /* the block's length */
    uc x4[16];
    uc name[8];
    uc x5[12];
} daveBlockInfo;
/**
    PDU handling:
    PDU is the central structure present in S7 communication.
    It is composed of a 10 or 12 byte header,a parameter block and a data block.
    When reading or writing values, the data field is itself composed of a data
    header followed by payload data
**/
typedef struct {
    uc P;	/* allways 0x32 */
    uc type;	/* Header type, one of 1,2,3 or 7. type 2 and 3 headers are two bytes longer. */
    uc a,b;	/* currently unknown. Maybe it can be used for long numbers? */
    us number;	/* A number. This can be used to make sure a received answer */
		/* corresponds to the request with the same number. */
    us plen;	/* length of parameters which follow this header */
    us dlen;	/* length of data which follow the parameters */
    uc result[2]; /* only present in type 2 and 3 headers. This contains error information. */
} PDUHeader;

/*
    same as above, but made up of single bytes only, so that every single byte can be adressed separately
*/
typedef struct {
    uc P;	/* allways 0x32 */
    uc type;	/* Header type, one of 1,2,3 or 7. type 2 and 3 headers are two bytes longer. */
    uc a,b;	/* currently unknown. Maybe it can be used for long numbers? */
    uc numberHi,numberLo;	/* A number. This can be used to make sure a received answer */
		/* corresponds to the request with the same number. */
    uc plenHi,plenLo;	/* length of parameters which follow this header */
    uc dlenHi,dlenLo;	/* length of data which follow the parameters */
    uc result[2]; /* only present in type 2 and 3 headers. This contains error information. */
} PDUHeader2;
/*
    set up the header. Needs valid header pointer in the struct p points to.
*/
EXPORTSPEC void DECL2 _daveInitPDUheader(PDU * p, int type);
/*
    add parameters after header, adjust pointer to data.
    needs valid header
*/
EXPORTSPEC void DECL2 _daveAddParam(PDU * p,uc * param,us len);
/*
    add data after parameters, set dlen
    needs valid header,and valid parameters.
*/
EXPORTSPEC void DECL2 _daveAddData(PDU * p,void * data,int len);
/*
    add values after value header in data, adjust dlen and data count.
    needs valid header,parameters,data,dlen
*/
EXPORTSPEC void DECL2 _daveAddValue(PDU * p,void * data,int len);
/*
    add data in user data. Add a user data header, if not yet present.
*/
EXPORTSPEC void DECL2 _daveAddUserData(PDU * p, uc * da, int len);
/*
    set up pointers to the fields of a received message
*/
EXPORTSPEC int DECL2 _daveSetupReceivedPDU(daveConnection * dc,PDU * p);
/*
    Get the eror code from a PDU, if one.
*/
EXPORTSPEC int DECL2 daveGetPDUerror(PDU * p);
/*
    send PDU to PLC and retrieve the answer
*/
EXPORTSPEC int DECL2 _daveExchange(daveConnection * dc,PDU *p);
/*
    retrieve the answer
*/
EXPORTSPEC int DECL2 daveGetResponse(daveConnection * dc);
/*
    send PDU to PLC
*/
EXPORTSPEC int DECL2 daveSendMessage(daveConnection * dc, PDU * p);
```

