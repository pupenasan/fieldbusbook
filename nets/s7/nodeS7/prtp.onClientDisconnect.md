[<-- До опису бібліотеки](README.md) 

## onClientDisconnect

```js
NodeS7.prototype.onClientDisconnect = function() {
	var self = this;
	outputLog('ISO-on-TCP connection DISCONNECTED.', 0, self.connectionID);

	// We issue the callback here for Trela/Honcho - in some cases TCP connects, 
    // and ISO-on-TCP doesn't.
	// If this is the case we need to issue the Connect CB in order to keep trying.
	if ((!self.connectCBIssued) && (typeof (self.connectCallback) === "function")) {
		self.connectCBIssued = true;
		self.connectCallback("Error - TCP connected, ISO didn't");
	}

	// We used to call self.connectionCleanup() - in other words we would give up.
	// However - realize that this event is called when the OTHER END 
    // of the connection sends a FIN packet.
	// Certain situations (download user program to mem card on S7-400, 
    // pop memory card out of S7-300, both with NetLink) cause this to happen.
	// So now, let's try a "connetionReset".  This way, we are guaranteed to return
    // values (or bad) and reset at the proper time.
	// self.connectionCleanup();
	self.connectionReset();
}
```





[<-- До опису бібліотеки](README.md) 





