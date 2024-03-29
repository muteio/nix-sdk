# nix-sdk
A modern NIX Core REST and RPC client to execute administrative tasks, [multiwallet](https://bitcoincore.org/en/2017/09/01/release-0.15.0/#multiwallet) operations and queries about network and the blockchain.

## Status
[![npm version][npm-image]][npm-url] [![build status][travis-image]][travis-url]

## Installation

Install the package via `yarn`:

```sh
yarn add nix-sdk
```

or via `npm`:

Install the package via `npm`:

```sh
npm install nix-sdk --save
```

## Usage
### Client(...args)
#### Arguments
1. `[agentOptions]` _(Object)_: Optional `agent` [options](https://github.com/request/request#using-optionsagentoptions) to configure SSL/TLS.
2. `[headers=false]` _(boolean)_: Whether to return the response headers.
3. `[host=localhost]` _(string)_: The host to connect to.
4. `[logger=debugnyan('nix-sdk')]` _(Function)_: Custom logger (by default, `debugnyan`).
5. `[network=mainnet]` _(string)_: The network
6. `[password]` _(string)_: The RPC server user password.
7. `[port=[network]]` _(string)_: The RPC server port (nix default: 6214).
8. `[ssl]` _(boolean|Object)_: Whether to use SSL/TLS with strict checking (_boolean_) or an expanded config (_Object_).
9. `[ssl.enabled]` _(boolean)_: Whether to use SSL/TLS.
10. `[ssl.strict]` _(boolean)_: Whether to do strict SSL/TLS checking (certificate must match host).
11. `[timeout=30000]` _(number)_: How long until the request times out (ms).
12. `[username]` _(number)_: The RPC server user name.
13. `[wallet]` _(string)_: Which wallet to manage ([read more](#multiwallet)).

### Examples
#### Using network mode
The `network` will automatically determine the port to connect to, just like the `nixd` and `nix-cli` commands.

```js
const Client = require('nix-sdk');
const client = new Client({ network: 'mainnet' });
```

##### Setting a custom port

```js
const client = new Client({ port: 28332 });
```

#### Connecting to an SSL/TLS server with strict checking enabled
By default, when `ssl` is enabled, strict checking is implicitly enabled.

```js
const fs = require('fs');
const client = new Client({
  agentOptions: {
    ca: fs.readFileSync('/etc/ssl/nixd/cert.pem')
  },
  ssl: true
});
```

#### Connecting to an SSL/TLS server without strict checking enabled

```js
const client = new Client({
  ssl: {
    enabled: true,
    strict: false
  }
});
```

#### Using promises to process the response

```js
client.getInfo().then((help) => console.log(help));
```

#### Using callbacks to process the response

```js
client.getInfo((error, help) => console.log(help));
```

#### Returning headers in the response
For compatibility with other NIX Core clients.

```js
const client = new Client({ headers: true });

// Promise style with headers enabled:
client.getInfo().then(([body, headers]) => console.log(body, headers));

// Await style based on promises with headers enabled:
const [body, headers] = await client.getInfo();
```

### Floating point number precision in JavaScript

Due to [JavaScript's limited floating point precision](http://floating-point-gui.de/), all big numbers (numbers with more than 15 significant digits) are returned as strings to prevent precision loss. This includes both the RPC and REST APIs.

## Multiwallet

This enables use-cases such as managing a personal and a business wallet simultaneously in order to simplify accounting and accidental misuse of funds.

To enable Multi Wallet support, start by specifying the number of added wallets you would like to have available and loaded on the server using the `-wallet` argument multiple times.

Notice the `rpcauth` hash which has been previously generated for the password `j1DuzF7QRUp-iSXjgewO9T_WT1Qgrtz_XWOHCMn_O-Y=`. Do **not** copy and paste this hash **ever** beyond this exercise.

Instantiate a client for each wallet and execute commands targeted at each wallet:

```js
const Client = require('nix-sdk');

const wallet1 = new Client({
  network: 'regtest',
  wallet: 'wallet1.dat',
  username: 'foo',
  password: 'j1DuzF7QRUp-iSXjgewO9T_WT1Qgrtz_XWOHCMn_O-Y='
});

const wallet2 = new Client({
  network: 'regtest',
  wallet: 'wallet2.dat',
  username: 'foo',
  password: 'j1DuzF7QRUp-iSXjgewO9T_WT1Qgrtz_XWOHCMn_O-Y='
});

(async function() {
  await wallet2.generate(100);

  console.log(await wallet1.getBalance());
  // => 0
  console.log(await wallet2.getBalance());
  // => 50
}());
```

### RPC
Start the `nixd` with the RPC server enabled and optionally configure a username and password:


These configuration values may also be set on the `nix.conf` file of your platform installation.

By default, port `6214` is used to listen for requests in `mainnet` mode, or `16214` in `testnet` and `regtest` modes. Use the `network` property to initialize the client on the desired mode and automatically set the respective default port. You can optionally set a custom port of your choice too.

The RPC services binds to the localhost loopback network interface, so use `rpcbind` to change where to bind to and `rpcallowip` to whitelist source IP access.

An example `nix.conf` may look like this:
```
server=1
par=1
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
rpcport=3335
rpcuser=username1
rpcpassword=password1
rpcclienttimeout=30
rpcthreads=2
rpcworkqueue=1000
staking=0
enableaccounts=1
```

#### Methods
All RPC [methods](src/methods.js) are exposed on the client interface as a camelcase'd version of those available on `nixd` (see examples below). You can also issue any command you desire by using the `command` command. Example for un-ghosting: `nix.command('unghostamountv2', '1');`

For a more complete reference about which methods are available, especially those unique to NIX, check the [commands documentation on the NIX wiki](https://wiki.nixplatform.io/home/wallet-functionality/console-commands).

##### Examples

#### Full ghosting and un-ghosting example

```js
/*
    ======================
    used nix.conf:

    server=1
    par=1
    rpcbind=127.0.0.1
    rpcallowip=127.0.0.1
    rpcport=3335
    rpcuser=username1
    rpcpassword=password1
    rpcclienttimeout=30
    rpcthreads=2
    rpcworkqueue=1000
    staking=0
    enableaccounts=1
    ======================
 */

const Client = require('nix-sdk');
const client = new Client({
    username: 'username1',
    password: 'password1',
    port: 3335,
    network: 'mainnet' });

client.command('ghostamountv2', '0.1').then(value => console.log(value)); //ghost 0.1 NIX and log response
client.command('unghostamountv2', '0.1').then(value => console.log(value)); //un-ghost 0.1 NIX and log response
```

#### Other example commands

```js 
client.command('unghostamount', '1');
client.createRawTransaction([{ txid: '1eb590cd06127f78bf38ab4140c4cdce56ad9eb8886999eb898ddf4d3b28a91d', vout: 0 }], { 'mgnucj8nYqdrPFh2JfZSB1NmUThUGnmsqe': 0.13 });
client.sendMany('test1', { mjSk1Ny9spzU2fouzYgLqGUD8U41iR35QN: 0.1, mgnucj8nYqdrPFh2JfZSB1NmUThUGnmsqe: 0.2 }, 6, 'Example Transaction');
client.sendToAddress('mmXgiR6KAhZCyQ8ndr2BCfEq1wNG2UnyG6', 0.1,  'sendtoaddress example', 'Nemo From Example.com');
```

#### Batch requests
Batch requests are support by passing an array to the `command` method with a `method` and optionally, `parameters`. The return value will be an array with all the responses.

```js
const batch = [
  { method: 'getnewaddress', parameters: [] },
  { method: 'getnewaddress', parameters: [] }
]

new Client().command(batch).then((responses) => console.log(responses)));

// Or, using ES2015 destructuring.
new Client().command(batch).then(([firstAddress, secondAddress]) => console.log(firstAddress, secondAddress)));
```

Note that batched requests will only throw an error if the batch request itself cannot be processed. However, each individual response may contain an error akin to an individual request.

```js
const batch = [
  { method: 'foobar', params: [] },
  { method: 'getnewaddress', params: [] }
]

new Client().command(batch).then(([address, error]) => console.log(address, error)));
// => `mkteeBFmGkraJaWN5WzqHCjmbQWVrPo5X3, { [RpcError: Method not found] message: 'Method not found', name: 'RpcError', code: -32601 }`.
```

### REST
Support for the REST interface is still **experimental** and the API is still subject to change. These endpoints are also **unauthenticated** so [there are certain risks which you should be aware](https://github.com/nixplatform/nixcore/blob/master/doc/REST-interface.md#risks), specifically of leaking sensitive data of the node if not correctly protected.

Error handling is still fragile so avoid passing user input.

Start the `nixd` with the REST server enabled:

These configuration values may also be set on the `nix.conf` file of your platform installation. Use `txindex=1` if you'd like to enable full transaction query support (note: this will take a considerable amount of time on the first run).

### Methods

Unlike RPC methods which are automatically exposed on the client, REST ones are handled individually as each method has its own specificity. The following methods are supported:

- [getBlockByHash](#getblockbyhashhash-options-callback)
- [getBlockHeadersByHash](#getblockheadersbyhashhash-count-options-callback)
- [getBlockchainInformation](#getblockchaininformationcallback)
- [getMemoryPoolContent](#getmemorypoolcontent)
- [getMemoryPoolInformation](#getmemorypoolinformationcallback)
- [getTransactionByHash](#gettransactionbyhashhash-options-callback)
- [getUnspentTransactionOutputs](#getunspenttransactionoutputsoutpoints-options-callback)

#### getBlockByHash(hash, [options], [callback])
Given a block hash, returns a block, in binary, hex-encoded binary or JSON formats.

##### Arguments
1. `hash` _(string)_: The block hash.
2. `[options]` _(Object)_: The options object.
3. `[options.extension=json]` _(string)_: Return in binary (`bin`), hex-encoded binary (`hex`) or JSON (`json`) format.
4. `[callback]` _(Function)_: An optional callback, otherwise a Promise is returned.

##### Example

```js
client.getBlockByHash('0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206', { extension: 'json' });
```

#### getBlockHeadersByHash(hash, count, [options][, callback])
Given a block hash, returns amount of block headers in upward direction.

##### Arguments
1. `hash` _(string)_: The block hash.
2. `count` _(number)_: The number of blocks to count in upward direction.
3. `[options]` _(Object)_: The options object.
4. `[options.extension=json]` _(string)_: Return in binary (`bin`), hex-encoded binary (`hex`) or JSON (`json`) format.
5. `[callback]` _(Function)_: An optional callback, otherwise a Promise is returned.

##### Example

```js
client.getBlockHeadersByHash('0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206', 1, { extension: 'json' });
```

#### getBlockchainInformation([callback])
Returns various state info regarding block chain processing.

#### Arguments
1. `[callback]` _(Function)_: An optional callback, otherwise a Promise is returned.

##### Example

```js
client.getBlockchainInformation([callback]);
```

#### getMemoryPoolContent()
Returns transactions in the transaction memory pool.

#### Arguments
1. `[callback]` _(Function)_: An optional callback, otherwise a Promise is returned.

##### Example

```js
client.getMemoryPoolContent();
```

#### getMemoryPoolInformation([callback])
Returns various information about the transaction memory pool. Only supports JSON as output format.
- size: the number of transactions in the transaction memory pool.
- bytes: size of the transaction memory pool in bytes.
- usage: total transaction memory pool memory usage.

#### Arguments
1. `[callback]` _(Function)_: An optional callback, otherwise a Promise is returned.

##### Example

```js
client.getMemoryPoolInformation();
```

#### getTransactionByHash(hash, [options], [callback])
Given a transaction hash, returns a transaction in binary, hex-encoded binary, or JSON formats.

#### Arguments
1. `hash` _(string)_: The transaction hash.
2. `[options]` _(Object)_: The options object.
3. `[options.summary=false]` _(boolean)_: Whether to return just the transaction hash, thus saving memory.
4. `[options.extension=json]` _(string)_: Return in binary (`bin`), hex-encoded binary (`hex`) or JSON (`json`) format.
5. `[callback]` _(Function)_: An optional callback, otherwise a Promise is returned.

##### Example

```js
client.getTransactionByHash('b4dd08f32be15d96b7166fd77afd18aece7480f72af6c9c7f9c5cbeb01e686fe', { extension: 'json', summary: false });
```

#### getUnspentTransactionOutputs(outpoints, [options], [callback])
Query unspent transaction outputs (UTXO) for a given set of outpoints. See [BIP64](https://github.com/bitcoin/bips/blob/master/bip-0064.mediawiki) for input and output serialisation.

#### Arguments
1. `outpoints` _(array\<Object\>|Object)_: The outpoint to query in the format `{ id: '<txid>', index: '<index>' }`.
2. `[options]` _(Object)_: The options object.
3. `[options.extension=json]` _(string)_: Return in binary (`bin`), hex-encoded binary (`hex`) or JSON (`json`) format.
4. `[callback]` _(Function)_: An optional callback, otherwise a Promise is returned.

##### Example

```js
client.getUnspentTransactionOutputs([{
  id: '0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206',
  index: 0
}, {
  id: '0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206',
  index: 1
}], { extension: 'json' }, [callback])
```

### SSL
This client supports SSL out of the box. Simply pass the SSL public certificate to the client and optionally disable strict SSL checking which will bypass SSL validation (the connection is still encrypted but the server it is connecting to may not be trusted). This is, of course, discouraged unless for testing purposes when using something like self-signed certificates.

#### Generating a self-signed certificates for testing purposes
Please note that the following procedure should only be used for testing purposes.

Generate an self-signed certificate together with an unprotected private key:

```sh
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 3650 -nodes
```

#### Connecting via SSL

You must use `stunnel` (`brew install stunnel` or `sudo apt-get install stunnel4`) or an HTTPS reverse proxy to configure SSL since the built-in support for SSL has been removed. The trade off with `stunnel` is performance and simplicity versus features, as it lacks more powerful capacities such as Basic Authentication and caching which are standard in reverse proxies.

You can use `stunnel` by configuring `stunnel.conf` with the following service requirements:

```
[nix]
accept = 28332
connect = 18332
cert = /etc/ssl/nixd/cert.pem
key = /etc/ssl/nixd/key.pem
```

The `key` option may be omitted if you concatenating your private and public certificates into a single `stunnel.pem` file.

On some versions of `stunnel` it is also possible to start a service using command line arguments. The equivalent would be:

```sh
stunnel -d 28332 -r 127.0.0.1:18332 -p stunnel.pem -P ''
```

Then pass the public certificate to the client:

```js
const Client = require('nix-sdk');
const fs = require('fs');
const client = new Client({
  agentOptions: {
    ca: fs.readFileSync('/etc/ssl/nixd/cert.pem')
  },
  port: 28332,
  ssl: true
});
```

## Logging

By default, all requests made with `nix-sdk` are logged using [uphold/debugnyan](https://github.com/uphold/debugnyan) with `nix-sdk` as the logging namespace.

Please note that all sensitive data is obfuscated before calling the logger.

#### Example

Example output defining the environment variable `DEBUG=nix-sdk`:

```javascript
const client = new Client();

client.getTransactionByHash('b4dd08f32be15d96b7166fd77afd18aece7480f72af6c9c7f9c5cbeb01e686fe');

// {
//   "name": "nix-sdk",
//   "hostname": "localhost",
//   "pid": 57908,
//   "level": 20,
//   "request": {
//     "headers": {
//       "host": "localhost:8332",
//       "accept": "application/json"
//     },
//     "id": "82cea4e5-2c85-4284-b9ec-e5876c84e67c",
//     "method": "GET",
//     "type": "request",
//     "uri": "http://localhost:8332/rest/tx/b4dd08f32be15d96b7166fd77afd18aece7480f72af6c9c7f9c5cbeb01e686fe.json"
//   },
//   "msg": "Making request 82cea4e5-2c85-4284-b9ec-e5876c84e67c to GET http://localhost:8332/rest/tx/b4dd08f32be15d96b7166fd77afd18aece7480f72af6c9c7f9c5cbeb01e686fe.json",
//   "time": "2017-02-07T14:40:35.020Z",
//   "v": 0
// }
```

### Custom logger

A custom logger can be passed via the `logger` option and it should implement [bunyan's log levels](https://github.com/trentm/node-bunyan#levels).

## Tests
Currently the test suite is tailored for Docker (including `docker-compose`) due to the multitude of different `nixd` configurations that are required in order to get the test suite passing.

To test using a local installation of `node.js` but with dependencies (e.g. `nixd`) running inside Docker:

```sh
npm run dependencies
npm test
```

To test using Docker exclusively (similarly to what is done in Travis CI):

```sh
npm run testdocker
```

## Release

```sh
npm version [<newversion> | major | minor | patch] -m "Release %s"
```

## License
MIT
