## cleanup: Use the `-rpcwait` when waiting for bitcoind to warm up *#3505*

We currently use a rather lengthy loop to wait for bitcoind to warm up and be ready to serve RPC calls:

```c
for (;;) { 
 		child = pipecmdarr(NULL, &from, &from, cast_const2(char **,cmd)); 
 		if (child < 0) { 
 			if (errno == ENOENT) 
 				bitcoind_failure(p, "bitcoin-cli not found. Is bitcoin-cli " 
 						    "(part of Bitcoin Core) available in your PATH?"); 
 			plugin_err(p, "%s exec failed: %s", cmd[0], strerror(errno)); 
 		} 
  
 		char *output = grab_fd(cmd, from); 
  
 		while ((ret = waitpid(child, &status, 0)) < 0 && errno == EINTR); 
 		if (ret != child) 
 			bitcoind_failure(p, tal_fmt(bitcoind, "Waiting for %s: %s", 
 						    cmd[0], strerror(errno))); 
 		if (!WIFEXITED(status)) 
 			bitcoind_failure(p, tal_fmt(bitcoind, "Death of %s: signal %i", 
 						   cmd[0], WTERMSIG(status))); 
  
 		if (WEXITSTATUS(status) == 0) 
 			break; 
  
 		/* bitcoin/src/rpc/protocol.h: 
 		 *	RPC_IN_WARMUP = -28, //!< Client still warming up 
 		 */ 
 		if (WEXITSTATUS(status) != 28) { 
 			if (WEXITSTATUS(status) == 1) 
 				bitcoind_failure(p, "Could not connect to bitcoind using" 
 						    " bitcoin-cli. Is bitcoind running?"); 
 			bitcoind_failure(p, tal_fmt(bitcoind, "%s exited with code %i: %s", 
 						    cmd[0], WEXITSTATUS(status), output)); 
 		} 
  
 		if (!printed) { 
 			plugin_log(p, LOG_UNUSUAL, 
 				   "Waiting for bitcoind to warm up..."); 
 			printed = true; 
 		} 
 		sleep(1); 
 	} 
```

This could be simplified if we just call bitcoin-cli with -rpcwait which implements the wait logic internally and allows us to skip checking the returned error (if we get an error it's not the warmup error so we can pass any error up regardless). The downside is that it doesn't allow us to print the warmup ourselves, but we could easily replace that with a single one-shot timer printing the message.









Source:

https://github.com/bitcoin/bitcoin/blob/master/doc/build-osx.md

https://github.com/ElementsProject/lightning/blob/44d408cc87beb854bdaf60c72108bb4492cfb34b/doc/INSTALL.md#to-build-on-macos

https://github.com/bitcoin/bitcoin/blob/master/share/examples/bitcoin.conf

https://stackoverflow.com/questions/20465039/what-does-wexitstatusstatus-return

https://github.com/bitcoin/bitcoin/blob/master/src/rpc/protocol.h

https://github.com/bitcoin/bitcoin/edit/master/share/examples/bitcoin.conf

https://github.com/ElementsProject/lightning/pull/3488/commits/44d408cc87beb854bdaf60c72108bb4492cfb34b



