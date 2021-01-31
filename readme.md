# Lightning Network

## Résolution d'une `first good issue`

### IF-4
**Henri Marteville  
Antoine Thibault  
Louis Tiercery**

---

## cleanup: Use the -rpcwait when waiting for bitcoind to warm up #3505
https://github.com/ElementsProject/lightning/issues/3505

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

---

## Etape 0: Description de la fonction impliqué
*Le but de cette fonction de s'assurer que bitcoind est disponible et qu'il est en mesure de recevoir des requêtes RPC de bitcoin-cli. Pour ce faire, la fonction essaye d'exécuter la commande `bitcoin-cli [args] getnetworkinfo`. Si le retour de cette commande est bien une variable json que l'on peut parser, alors on est assurer que bitcoind est prêt pour la suite de lightning.*
```c
static void wait_and_check_bitcoind(struct plugin *p)
{
	int from, status, ret;
	pid_t child;
	const char **cmd = gather_args(bitcoind, "getnetworkinfo", NULL);
	bool printed = false;
	char *output = NULL;

	for (;;) {
		tal_free(output);

		child = pipecmdarr(NULL, &from, &from, cast_const2(char **,cmd));
		if (child < 0) {
			if (errno == ENOENT)
				bitcoind_failure(p, "bitcoin-cli not found. Is bitcoin-cli "
						    "(part of Bitcoin Core) available in your PATH?");
			plugin_err(p, "%s exec failed: %s", cmd[0], strerror(errno));
		}

		output = grab_fd(cmd, from);

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

	parse_getnetworkinfo_result(p, output);

	tal_free(cmd);
}
```

## Etape 1: trigger les erreurs que le code est capable de détecter


```c
child = pipecmdarr(NULL, &from, &from, cast_const2(char **,cmd)); // run a command, optionally connect pipes (char arry version)
 		if (child < 0) { 
 			if (errno == ENOENT) 
 				bitcoind_failure(p, "bitcoin-cli not found. Is bitcoin-cli " 
 						    "(part of Bitcoin Core) available in your PATH?"); 
 			plugin_err(p, "%s exec failed: %s", cmd[0], strerror(errno)); 
 		}
```
Ce morceau de code execute la commande constitué à partir des arguments rassemblés par cette instruction: `const char **cmd = gather_args(bitcoind, "getnetworkinfo", NULL);`

Dans un conteneur docker ubuntu, on install seulement lightning (sans 
```
docker run -ti ubuntu
apt-get update
apt-get install -y autoconf automake build-essential git libtool libgmp-dev libsqlite3-dev python3 python3-mako net-tools zlib1g-dev libsodium-dev gettext
git clone https://github.com/ElementsProject/lightning.git
cd lightning
./configure
make
```

`error returned`:
```shell script
bitcoin-cli not found. Is bitcoin-cli (part of Bitcoin Core) available in your PATH?

Make sure you have bitcoind running and that bitcoin-cli is able to connect to bitcoind.

You can verify that your Bitcoin Core installation is ready for use by running:

    $ bitcoin-cli echo 'hello world'

2021-01-30T06:19:08.700Z INFO    plugin-bcli: Killing plugin: Plugin exited before completing handshake.
The Bitcoin backend died.
```
Cette erreur apparaitra même si l'on rajoute la commande -rpcwait. En effet, puisqu'il s'agit d'une commande de bitcoin-cli, si celui-ci n'est pas accessible ou n'est pas installé, l'erreur se produira. **Donc on doit garder la logique de ce bout de code.**

---

```c
while ((ret = waitpid(child, &status, 0)) < 0 && errno == EINTR);
		if (ret != child)
			bitcoind_failure(p, tal_fmt(bitcoind, "Waiting for %s: %s",
						    cmd[0], strerror(errno)));
		if (!WIFEXITED(status))
			bitcoind_failure(p, tal_fmt(bitcoind, "Death of %s: signal %i",
						   cmd[0], WTERMSIG(status)));
```         
Ce morceau de code nous indique si le process de bitcoin-cli est accessible. 
Dans un premier temps, si  ret = waitpid < 0 (waitpid est un appel qui suspend l'exécution du processus appelant jusqu'à ce qu'un  fils spécifié  par  l'argument  pid  change d'état) et errno == EINTR (EINTR si un signal s'est produit pendant que l'appel système était en cours), alors : 
	- si ret n'est pas egal à child, comme la variable child a été initialisée en faisant appel à pipecmdarr alors cela veut dire que le processus bitcoind n'a pas encore changé d'état et que donc il est en train de se lancer. Le programme renvoie bitcoind_failure qui précise que lnd attend que bitcoind se lance. 
	- si bitcoind ne renvoie pas WIFEXITED(status) (qui renvoie true si le fils s'est terminé correctement, c'est à dire par un appel à exit() __exit() ou un retour de main()) cela signifie que le signal est mort et donc le programme renvoie bitcoind_failure et précise que le signal bitcoind est mort. 
Si rien de cela n'est arrivé, alors quand bitcoind sera demarré il changera d'état, waitpid deviendrea > 0 et le programme peux continuer. 

---

```c
if (WEXITSTATUS(status) == 0)
			break;
```   
Ce morceau de code nous fait sortir de la boucle car il indique que le process bitcoin-cli est disponible et opérationnel.

---

```c
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
```
Ce morceau de code est interressant car c'est la que vient l'intérêt de la boucle. En effet, le process bitcoind peut être actif et pourtant ne pas être capable de recevoir des requêtes. Cette phase dure assez longtemps et se débloque en même temps que bitcoind commence à télécharger des blocs. Ce que cela veut aussi dire, c'est que le process de bitcoin-cli peut être actif et pourtant ne pas être en mesure de faire la commande car bitcoind est toujours en train de warm up. Le status de bitcoin-cli est arbitrairement 28.  
Donc tant que bitcoin-cli nous retourne un status et que celui-ci est 28, on continue la boucle jusqu'à ce qu'on tombe sur le status 0 présenté précdemment qui indique un fonctionnement normal.
Si le status est différent de 28, on regarde les autres valeurs qui peuvent inquider un mauvais fonctionnement. Pour ce qui est de simplement attendre que bitcoind soit prêt à recevoir des requêtes RPC, seul le status 1 est révélateur d'une impossibilité de dialogue entre bitcoin-cli et bitcoind. Cela peut simplement vouloir dire que le process de bitcoind n'est pas actif. On le verifira par l'exemple juste après.

```shell script
//! General application defined errors
    RPC_MISC_ERROR                  = -1,  //!< std::exception thrown in command handling
    RPC_TYPE_ERROR                  = -3,  //!< Unexpected type was passed as parameter
    RPC_INVALID_ADDRESS_OR_KEY      = -5,  //!< Invalid address or key
    RPC_OUT_OF_MEMORY               = -7,  //!< Ran out of memory during operation
    RPC_INVALID_PARAMETER           = -8,  //!< Invalid, missing or duplicate parameter
    RPC_DATABASE_ERROR              = -20, //!< Database error
    RPC_DESERIALIZATION_ERROR       = -22, //!< Error parsing or validating structure in raw format
    RPC_VERIFY_ERROR                = -25, //!< General error during transaction or block submission
    RPC_VERIFY_REJECTED             = -26, //!< Transaction or block was rejected by network rules
    RPC_VERIFY_ALREADY_IN_CHAIN     = -27, //!< Transaction already in chain
    RPC_IN_WARMUP                   = -28, //!< Client still warming up
    RPC_METHOD_DEPRECATED           = -32, //!< RPC method is deprecated 
```

`error returned`:
```shell script
Could not connect to bitcoind using bitcoin-cli. Is bitcoind running?

Make sure you have bitcoind running and that bitcoin-cli is able to connect to bitcoind.

You can verify that your Bitcoin Core installation is ready for use by running:

    $ bitcoin-cli -testnet -datadir=/Volumes/ETH/bitcoin echo 'hello world'

2021-01-30T07:24:41.654Z INFO    plugin-bcli: Killing plugin: Plugin exited before completing handshake.
The Bitcoin backend died.
```
Si les exécutables bitcoind et bitcoin-cli sont disponible, mais que le process de bitcoind n'est pas actif, alors la commande bitcoin-cli [args] getnetworkinfo retourne une erreur, ce qui stop la boucle.

## 2 eme étape: implémenter -rpcwait

En effet, si on essaye d'executer la commande suivante, on obtient l'erreur qui suit:
```shell script
~ bitcoin-cli -testnet -datadir=/Volumes/ETH/bitcoin getnetworkinfo 
error: Could not connect to the server 127.0.0.1:8332

Make sure the bitcoind server is running and that you are connecting to the correct RPC port.
```

---

Maintentant, si l'on essaye de la même commande avec l'argument -rpcwait en plus:
```
 ~ bitcoin-cli -testnet -datadir=/Volumes/ETH/bitcoin -rpcwait getnetworkinfo
```
On n'obtient rien, la commande ne retourne aucun status et n'exit pas. Maintenant, on peut remplacer cette commande par celle qui est utilisé dans la fonction et ainsi remplacer une grande partie de la logique mis en place. On garde garde donc la logique qui nous permet de dire si bitcoinc-cli est accessible et on le laisse effectue la commande et on attend que celui-ci réponde. Lorsque l'on obitent l'output, on le parse avec la fonction `parse_getnetworkinfo_result` pour s'assurer de la bonne santé de bitcoin-cli et on laisse ensuite le code se dérouler.

On rajoute l'argument `-rpcwait` dans la fonction gather_args:
```c
static const char **gather_args(const tal_t *ctx, const char *cmd, const char **cmd_args)
{
	const char **args = tal_arr(ctx, const char *, 1);

	args[0] = bitcoind->cli ? bitcoind->cli : chainparams->cli;
	if (chainparams->cli_args)
		add_arg(&args, chainparams->cli_args);
	if (bitcoind->datadir)
		add_arg(&args, tal_fmt(args, "-datadir=%s", bitcoind->datadir));
	if (bitcoind->rpcconnect)
		add_arg(&args,
			tal_fmt(args, "-rpcconnect=%s", bitcoind->rpcconnect));
	if (bitcoind->rpcport)
		add_arg(&args,
			tal_fmt(args, "-rpcport=%s", bitcoind->rpcport));
	if (bitcoind->rpcuser)
		add_arg(&args, tal_fmt(args, "-rpcuser=%s", bitcoind->rpcuser));
	if (bitcoind->rpcpass)
		add_arg(&args,
			tal_fmt(args, "-rpcpassword=%s", bitcoind->rpcpass));
	
	add_arg(&args, "-rpcwait"); // <- ici
	add_arg(&args, cmd);
	for (size_t i = 0; i < tal_count(cmd_args); i++)
		add_arg(&args, cmd_args[i]);
	add_arg(&args, NULL);

	return args;
}
```
*Oui, il y a certainement une facon plus élégante de faire. Mais il semble compliquer d'implémenter l'argument dans la logique de la variable `ctx` et la variable `cmd_args` rassemble les arguments qui seront placer derrière la méthode appellé (pour nous `getnetworkinfo`) et notre argument n'aura alors aucun effet.*

```c
static void wait_and_check_bitcoind(struct plugin *p)
{
	int from;
	pid_t child;
	const char **cmd = gather_args(bitcoind, "getnetworkinfo", NULL);
	char *output = NULL;

	child = pipecmdarr(NULL, &from, &from, cast_const2(char **,cmd));
	if (child < 0) {
		if (errno == ENOENT)
			bitcoind_failure(p, "bitcoin-cli not found. Is bitcoin-cli "
					    "(part of Bitcoin Core) available in your PATH?");
		plugin_err(p, "%s exec failed: %s", cmd[0], strerror(errno));
	}

	output = grab_fd(cmd, from);
	parse_getnetworkinfo_result(p, output);

	tal_free(output);
	tal_free(cmd);
}
```

---

Maintentant, lorsque l'on exécute la commande suivante:
```shell script
./lightningd/lightningd --testnet --lightning-dir=/Volumes/ETH/lnd --bitcoin-datadir=/Volumes/ETH/bitcoin
```
- si bitcoin-cli n'est pas installé, on a l'erreur suivante:
```shell script
bitcoin-cli not found. Is bitcoin-cli (part of Bitcoin Core) available in your PATH?

Make sure you have bitcoind running and that bitcoin-cli is able to connect to bitcoind.

You can verify that your Bitcoin Core installation is ready for use by running:

    $ bitcoin-cli echo 'hello world'

2021-01-30T06:19:08.700Z INFO    plugin-bcli: Killing plugin: Plugin exited before completing handshake.
The Bitcoin backend died.
```
- si bitcoin-cli est accessible, mais que bitcoind n'est pas prêt à recevoir des requêtes RPC, on attend simplement.

## Désavantages de cette solution
On essayé de profiter au maximum de ce que pouvait nous offir l'argument `-rpcwait`: Le problème avec cette solution c'est que l'on ne voit pas de log s'afficher, puisque la logique du warm up est délégué à bitcoin-cli. Ce que l'on pourrait faire c'est rajouter du code qui nous permet de comprendre si bitcoind est actif:

```c
static void wait_and_check_bitcoind(struct plugin *p)
{
	int from;
	pid_t child;
	const char **cmd = gather_args(bitcoind, "getnetworkinfo", NULL);
	char *output = NULL;

	child = pipecmdarr(NULL, &from, &from, cast_const2(char **,cmd));
	if (child < 0) {
		if (errno == ENOENT)
			bitcoind_failure(p, "bitcoin-cli not found. Is bitcoin-cli "
					    "(part of Bitcoin Core) available in your PATH?");
		plugin_err(p, "%s exec failed: %s", cmd[0], strerror(errno));
	}

	output = grab_fd(cmd, from);
	while ((ret = waitpid(child, &status, 0)) < 0 && errno == EINTR);
	if (WEXITSTATUS(status) == 1)
				bitcoind_failure(p, "Could not connect to bitcoind using"
						    " bitcoin-cli. Is bitcoind running?");
			bitcoind_failure(p, tal_fmt(bitcoind, "%s exited with code %i: %s",
						    cmd[0], WEXITSTATUS(status), output));
	
	parse_getnetworkinfo_result(p, output);

	tal_free(output);
	tal_free(cmd);
}
```
Seulement si l'on fait ca, au final, on ne retire que peu de code, il y a donc un tradeoff entre avoir un code plus simple mais moins maitriser le status du process bitcoind, ou alors garder une logique un peu verbeuse et être capable de maitriser le status de bitcoind 

Lien du fork: [ici](https://github.com/LTRY/lightning/blob/master)
Lien du fichier bcli.c du fork [ici](https://github.com/LTRY/lightning/blob/master/plugins/bcli.c)


`SOURCE`:

https://github.com/bitcoin/bitcoin/blob/master/doc/build-osx.md

https://github.com/ElementsProject/lightning/blob/44d408cc87beb854bdaf60c72108bb4492cfb34b/doc/INSTALL.md#to-build-on-macos

https://github.com/bitcoin/bitcoin/blob/master/share/examples/bitcoin.conf

https://stackoverflow.com/questions/20465039/what-does-wexitstatusstatus-return

https://github.com/bitcoin/bitcoin/blob/master/src/rpc/protocol.h

https://github.com/bitcoin/bitcoin/edit/master/share/examples/bitcoin.conf

https://github.com/ElementsProject/lightning/pull/3488/commits/44d408cc87beb854bdaf60c72108bb4492cfb34b



