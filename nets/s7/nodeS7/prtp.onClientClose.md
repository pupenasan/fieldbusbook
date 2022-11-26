[<-- До опису бібліотеки](README.md) 

## onClientClose

```js
NodeS7.prototype.onClientClose = function() {
	var self = this;
    // clean up the connection now the socket has closed
		// We used to call self.connectionCleanup() here, but it caused problems.
		// However - realize that this event is also called when the OTHER END of the connection sends a FIN packet.
		// Certain situations (download user program to mem card on S7-400, pop memory card out of S7-300, both with NetLink) cause this to happen.
		// So now, let's try a "connetionReset".  This way, we are guaranteed to return values (even if bad) and reset at the proper time.
		// Without this, client applications had to be prepared for a read/write not returning.
	self.connectionReset();

    // initiate the callback stored by dropConnection
    if (self.dropConnectionCallback) {
        self.dropConnectionCallback();
        // prevent any possiblity of the callback being called twice
        self.dropConnectionCallback = null;
        // and cancel the timeout
        clearTimeout(self.dropConnectionTimer);
    }
}
```





[<-- До опису бібліотеки](README.md) 





