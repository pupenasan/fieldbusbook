# Getting started

## Using the test programs

### Intended purpose

**Libnodave** comes with a set of test programs. They shall serve the following purposes:

Provide the user with a demostration of what Libnodave can do.

Provide a quick test for compatibility with user's hardware and configuration.

Provide a source code example as a template for your own applications.

The debug output, obtained with debug option (-d), provides valuable informatioon in case Libnodave fails with some hardware..

The benchmark options let you measure the time needed for transfers of short and long data blocks.

#### Note on PASCAL source test programs:

Basic test programs are also available as Pascal sources. They are not as complete as their C coded counterparts. Their main purpose is to test the interface unit **nodave.pas**.

### Calling test programs

Just invoke the test programs without arguments.  They will print a list of possible arguments and options.

### Which test program for which Setup?

| CPU           | Connection                            | Generell Tests | Load Program into CPU |
| ------------- | ------------------------------------- | -------------- | --------------------- |
| S7 300 or 400 | Serial with MPI adapter cable         | testMPI        | testMPIload           |
| S7 200        | Serial with PPI adapter cable         | testPPI        | testPPIload           |
| S7 300 or 400 | Ethernet CP343/443                    | testISO_TCP    | testISO_TCPload       |
| S7 200        | Ethernet CP243                        | testISO_TCP -2 | testISO_TCPload -2    |
| S7 300 or 400 | Ethernet with IBH/MHJ-NetLink Gateway | testIBH        | testMPI_IBHload       |
| S7 200        | Ethernet with IBH/MHJ-NetLink Gateway | testPPI_IBH    | testPPI_IBHload       |

### Options to make test programs work with certain configurations:

#### MPI transport with MPI adapter:

--mpi=[number] Uses number as the MPI address of the PLC. Default is 2.

--local=[number] Uses number as the local MPI address of the adapter. Default is 0.

-2 Uses another variant of the MPI protocol. Try if you connect to adapter/PLC.

-[some numbers] Use dfferent MPI/Profibus speeds.

#### ISO over TCP transport with CP343/CP443:

--slot=[number] Uses number as the slot address of the PLC. Default rack 0, slot 2.

#### ISO over TCP transport with CP243:

-2 Fakes a MicroWin connection request. This is currently mandatory for CP243.  It will switch the CFG LED on.

### What can the test programs do?

If invoked with the connection as the only argument, all test programs will read some data from  the memory area of Flags (also known as Merkers).They try to read FD0,FD4 and FD8 as DWORDS and  FD12 as real.
Depending on the contents of this memory, the results may or may no seem  reasonable.The obtained values are just the same as if you would observe these variables in Step7  using the display formats signed,signed,signed and floating point.
 If you specify the option -w, the test programs increment the data, write it back to your PLC  (Attention! This changes PLC internal memory!) and read it again, thus demonstrating the effect of  changes.
 If you specify the option -c, the test programs write 0 to these memory locations.  (Attention! This changes PLC internal memory!). This is useful if the current memory contents doesn't make much sense when displayed in the above mentioned format.
 If you specify the option -b, the test programs try to do block read benchmark tests. Combining -b and  -w will also do write benchmark tests.
 -r option tries to put your PLC into RUN mode.(Attention! May start actions on machinery!).
 -s option tries to put your PLC into STOP mode.(Attention! Will interrupt running machinery!).
 --readout option tries to readout program blocks from the PLC and will store them in files named like OB1.mc7.(Attention! I got reports, that this produces ill effects on S7-400!).
 -z reads System State Lists (**S**ystem-**Z**ustands**l**isten from the PLC. These lists exist only in 300/400 family PLCs and provide diagnostic information. Please refer to Siemens documentation about the meaning of IDs and indices.

### Test programs to load blocks into CPU

This is a quite experimental feature.  First you will need correctly formed program blocks stored in files. A possible source are files previously read out using --readout. You cannot get them from Step7 as Step7 stores program block in a data base. It has not been tested but may be so that third party programming software  that stores program blocks in files uses the same file format.
 Loading of SDBs (**S**ystem **D**ata **B**locks) is highly dependent of the sequence of  block numbers.

## Programming a basic application

#### Preparations

The main purpose of this library is to read and write data from and to Siemens PLCs. To do so, you need to establish a connection to the PLC. First, you need to configure a serial port of your computer or establish a TCP connection. This connection is represented by the type _daveOSserialType, which contains file descriptors in case of Unix-like systems, handles in case of windows and what other systems supported in  the future might use for this purpose. use setport to initialize the members of a [_daveOSserialType](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/daveOSserialType.html) to something representing a configured serial connection:

```
    fds.rfd=setPort(argv[adrPos],"38400",'O');
```

for serial connections or:    

```
    fds.rfd=openSocket(102, IPaddress_of_CP);
```

or

```
    fds.rfd=openSocket(1099, IPaddress_of_IBH-NetLink);
```

for TCP connections. Then do:    

```
    fds.wfd=fds.rfd;
```

With the initialized _daveOSserialType, you will initialize a structure of type  [daveInterface](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/daveInterface.html),  representing the physical connection to a PLC or a network of PLCs (e.g. like MPI).

```
    di=daveNewInterface(fds, "IF1", localMPI, daveProtoXXX, daveSpeedYYY);
```

With the resulting daveInterface structure, you can initialize an adapter, if one is used:

```
    res =daveInitAdapter(di);
```

​     While currently only MPI-adapters and IBH-NetLinks really need this initialization procedure, it is save to use daveInitAdapter() with any protocol type. If it has no meaning for the protocol used, it is mapped to a dummy procedure that returns allways success. After successfully initializing your adapter, you can retrieve a list of reachable partners on an MPI network. The function takes the daveInterface structure and a pointer to a buffer of sufficient length as arguments. It returns the real length of the list. If the partners cannot be listed with the protocol used, it just returns a length of 0.

```
    listLength = daveListReachablePartners(di,buf1);
```

After successfully initializing your adapter, you can establish a connection to a certain PLC on the network. To do so, you will first initialize a structure of  type [daveConnection](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/daveConnection.html),  representing the logical connection to a single PLC.

```
    dc =daveNewConnection(di, MPI_address, Rack, Slot);
```

With the resulting daveConnection structure, you need to really connect the PLC:

```
    res =daveConnectPLC(dc);
```

#### Exchanging values:

Once you have established a connection to your PLC, you can read and write values:

```
    res=daveReadBytes(dc, AREA, area_Number, start_address, length, buffer);
    res=daveWriteBytes(dc, AREA, area_Number, start_address, length, buffer);
```

Usually, you will have to [convert](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/conversions.html) byte sequences from and to the buffer to use the data in your application.
 After you are done with your data exchanges, call:

```
	daveDisconnectPLC(dc);
```

To disconnect from the PLC and

```
	daveDisconnectAdapter(di);
```

​	 to disconnect from the Adapter. Now close the serial or TCP/IP connection using the appropriate system calls for your OS.

# Advanced data exchange

[Read multiple items with a single transaction.](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/readmultiple.html)
[Read and set single bits.](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/bitfunctions.html)

# Other features

[Read diagnostic info (300 and 400 only).](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/SZL.html)
[Load program code from PLC.](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/upload.html)
[Load program code into PLC.](file:///C:/Users/OleksandrPupena/Downloads/libnodave-0.8.5/doc/upload.html)