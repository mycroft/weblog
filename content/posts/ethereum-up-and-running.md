---
title:  "Ethereum: Up & Running"
date:   2018-04-01T08:48:14+0200
categories:
  - ethereum
summary: "Quick introduction to running Ethereum"
---

One of my recent jobs was to build a simple payment processor for [Ethereum](https://www.ethereum.org/). The task was way simpler I thought it would be, but starting to work with `Ethereum` can be a little confusing. Unlike `Bitcoin`, there is not only one single monolithic component like [Bitcoin Core](https://bitcoin.org/en/bitcoin-core/) to work with, but a few, like [geth](https://github.com/ethereum/go-ethereum/wiki/geth) - the CLI for running a full Ethereum node, or [mist](https://github.com/ethereum/mist), the wallet/browser for dapps.

## Installing geth

`geth` is a [golang](https://golang.org/) app, and it is very simple to install:

```shell
$ cd /opt
$ git clone https://github.com/ethereum/go-ethereum.git
Cloning into 'go-ethereum'...
remote: Counting objects: 69307, done.
remote: Total 69307 (delta 0), reused 0 (delta 0), pack-reused 69307
Receiving objects: 100% (69307/69307), 95.20 MiB | 8.63 MiB/s, done.
Resolving deltas: 100% (46104/46104), done.
$ cd go-ethereum
$ make all
[...]
github.com/ethereum/go-ethereum/cmd/geth
github.com/ethereum/go-ethereum/cmd/swarm
github.com/ethereum/go-ethereum/cmd/wnode
$ ls -l build/bin/
total 270568
[...]
-rwxr-xr-x. 1 root root 37394152 Apr  1 09:09 geth
[...]
$
```

And you're done. You can now launch geth to run a full `Ethereum` node on its main's network, on testnet or even create your own private chain, because unlike Bitcoin, it is very easy to create your own private network.

## Joining Ethereum main's network

Nothing much to say, just run `geth`. That's it.

```shell
$ /opt/go-ethereum/build/bin/geth 
INFO [04-01|09:17:02] Maximum peer count                       ETH=25 LES=0 total=25
INFO [04-01|09:17:02] Starting peer-to-peer node               instance=Geth/v1.8.4-unstable-a095b84e/linux-amd64/go1.10.1
INFO [04-01|09:17:02] Allocated cache and file handles         database=/home/mycroft/.ethereum/geth/chaindata cache=768 handles=512
INFO [04-01|09:17:02] Writing default main-net genesis block 
INFO [04-01|09:17:02] Persisted trie from memory database      nodes=12356 size=2.34mB time=37.468534ms gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [04-01|09:17:02] Initialised chain configuration          config="{ChainID: 1 Homestead: 1150000 DAO: 1920000 DAOSupport: true EIP150: 2463000 EIP155: 2675000 EIP158: 2675000 Byzantium: 4370000 Constantinople: <nil> Engine: ethash}"
INFO [04-01|09:17:02] Disk storage enabled for ethash caches   dir=/home/mycroft/.ethereum/geth/ethash count=3
INFO [04-01|09:17:02] Disk storage enabled for ethash DAGs     dir=/home/mycroft/.ethash               count=2
INFO [04-01|09:17:02] Initialising Ethereum protocol           versions="[63 62]" network=1
INFO [04-01|09:17:02] Loaded most recent local header          number=0 hash=d4e567…cb8fa3 td=17179869184
INFO [04-01|09:17:02] Loaded most recent local full block      number=0 hash=d4e567…cb8fa3 td=17179869184
INFO [04-01|09:17:02] Loaded most recent local fast block      number=0 hash=d4e567…cb8fa3 td=17179869184
INFO [04-01|09:17:02] Regenerated local transaction journal    transactions=0 accounts=0
INFO [04-01|09:17:02] Starting P2P networking 
INFO [04-01|09:17:04] UDP listener up                          self=enode://632f2cf51de6dafb0e1254177858069595706fde6b649fb7f4811f4a14066e8955a5407226a5f78463217527c40696846f6117e9297d3837882c823a11540e72@90.127.22.118:30303
INFO [04-01|09:17:04] RLPx listener up                         self=enode://632f2cf51de6dafb0e1254177858069595706fde6b649fb7f4811f4a14066e8955a5407226a5f78463217527c40696846f6117e9297d3837882c823a11540e72@90.127.22.118:30303
INFO [04-01|09:17:04] IPC endpoint opened                      url=/home/mycroft/.ethereum/geth.ipc
INFO [04-01|09:17:34] Block synchronisation started 
INFO [04-01|09:17:38] Imported new block headers               count=192 elapsed=894.468ms number=192 hash=723899…123390 ignored=0
INFO [04-01|09:17:38] Imported new block receipts              count=2   elapsed=127.337µs number=2   hash=b495a1…4698c9 size=8.00B  ignored=0
INFO [04-01|09:17:38] Imported new block headers               count=192 elapsed=17.487ms  number=384 hash=d3d5d5…c79cf3 ignored=0
INFO [04-01|09:17:38] Imported new state entries               count=523 elapsed=3.924µs   processed=523 pending=8366 retry=0 duplicate=0 unexpected=0
[...]
```

Now, you'll have to wait. Also, `Ethereum` is very CPU and I/O consumming (you'll need a SSD). If you don't need it, I strongly advise not to run an full Ethereum node.

## About Ethereum testnet

Contrary to `Bitcoin`, there is 3 Ethereum testnets: `ropsten`, `kovan` and `rinkeby`, and [they are all differents](https://ethereum.stackexchange.com/questions/27048/comparison-of-the-different-testnets) from the others.

  * Ropsten:
    * Proof of work (you can mine coins);
    * Supported by `geth` and [parity](https://github.com/paritytech/parity);
    * Reproduces the main net environment.
  * Rinkeby:
    * Proof of authority (can't mine new coins);
    * Supported by `geth` only.
  * Kovan:
    * Proof of authority;
    * Not supported by `geth`.

Joining a testnet network is as simple as joining mainnet. Just start `geth` using the `--testnet` switch to join `ropsten`, or the `--rinkeby` switch to join `rinkeby`:

```shell
$ /opt/go-ethereum/build/bin/geth --rinkeby
INFO [04-01|09:27:53] Maximum peer count                       ETH=25 LES=0 total=25
INFO [04-01|09:27:53] Starting peer-to-peer node               instance=Geth/v1.8.4-unstable-a095b84e/linux-amd64/go1.10.1
INFO [04-01|09:27:53] Allocated cache and file handles         database=/home/mycroft/.ethereum/rinkeby/geth/chaindata cache=768 handles=512
INFO [04-01|09:27:53] Writing custom genesis block 
INFO [04-01|09:27:53] Persisted trie from memory database      nodes=355 size=65.27kB time=1.459688ms gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [04-01|09:27:53] Initialised chain configuration          config="{ChainID: 4 Homestead: 1 DAO: <nil> DAOSupport: true EIP150: 2 EIP155: 3 EIP158: 3 Byzantium: 1035301 Constantinople: <nil> Engine: clique}"
INFO [04-01|09:27:53] Initialising Ethereum protocol           versions="[63 62]" network=4                                           
[...]
```

If you need coins on testnet networks, you can grab a few on the existing faucets. There is one for ropsten and one for [rinkeby](http://rinkeby-faucet.com/).


## And what's up with private chains ?

With Ethereum, you can create your own chain without having to patch source code, like `Bitcoin`. You just need a `genesis` file, a JSON configuration file that describes your chain.

Lets take a look at my last `genesis.json` file:

```json
{
    "config": {
        "chainId": 2283526326,
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
    },
    "difficulty": "0x200",
    "gasLimit": "0x8000000",
    "alloc": {
        "c97ec1b4bf2b0106f951e113690b194289037d52": { "balance": "1000000000000000000000000" }
    }
}
```

Create your chain using `geth init genesis.json`

```shell
$ geth --datadir .ethereum/sweet-refrain init genesis.json 
INFO [04-01|09:39:08] Maximum peer count                       ETH=25 LES=0 total=25
INFO [04-01|09:39:08] Allocated cache and file handles         database=/home/mycroft/.ethereum/sweet-refrain/geth/chaindata cache=16 handles=16
INFO [04-01|09:39:08] Writing custom genesis block 
INFO [04-01|09:39:08] Persisted trie from memory database      nodes=1 size=202.00B time=33.404µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [04-01|09:39:08] Successfully wrote genesis state         database=chaindata                                            hash=e82409…14d6b1
INFO [04-01|09:39:08] Allocated cache and file handles         database=/home/mycroft/.ethereum/sweet-refrain/geth/lightchaindata cache=16 handles=16
INFO [04-01|09:39:08] Writing custom genesis block 
INFO [04-01|09:39:08] Persisted trie from memory database      nodes=1 size=202.00B time=24.128µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [04-01|09:39:08] Successfully wrote genesis state         database=lightchaindata                                            hash=e82409…14d6b1
```

And launch your daemon:

```shell
$ /opt/go-ethereum/build/bin/geth --datadir .ethereum/sweet-refrain
INFO [04-01|09:40:25] Maximum peer count                       ETH=25 LES=0 total=25
INFO [04-01|09:40:25] Starting peer-to-peer node               instance=Geth/v1.8.4-unstable-a095b84e/linux-amd64/go1.10.1
INFO [04-01|09:40:25] Allocated cache and file handles         database=/home/mycroft/.ethereum/sweet-refrain/geth/chaindata cache=768 handles=512
INFO [04-01|09:40:25] Initialised chain configuration          config="{ChainID: 2283526326 Homestead: 0 DAO: <nil> DAOSupport: false EIP150: <nil> EIP155: 0 EIP158: 0 Byzantium: <nil> Constantinople: <nil> Engine: unknown}"
INFO [04-01|09:40:25] Disk storage enabled for ethash caches   dir=/home/mycroft/.ethereum/sweet-refrain/geth/ethash count=3
INFO [04-01|09:40:25] Disk storage enabled for ethash DAGs     dir=/home/mycroft/.ethash                             count=2
INFO [04-01|09:40:25] Initialising Ethereum protocol           versions="[63 62]" network=1
INFO [04-01|09:40:25] Loaded most recent local header          number=0 hash=e82409…14d6b1 td=512
INFO [04-01|09:40:25] Loaded most recent local full block      number=0 hash=e82409…14d6b1 td=512
INFO [04-01|09:40:25] Loaded most recent local fast block      number=0 hash=e82409…14d6b1 td=512
INFO [04-01|09:40:25] Loaded local transaction journal         transactions=0 dropped=0
INFO [04-01|09:40:25] Regenerated local transaction journal    transactions=0 accounts=0
INFO [04-01|09:40:25] Starting P2P networking
INFO [04-01|09:40:27] UDP listener up                          self=enode://bac80bb4ca615dfbfbf8a738169ac72b47c7917a53941458981c4f09f3e1c0c98e940091acd12d66bff1927d4d44a26ee07814faf27c3f63b628ebc83e44e0ec@[::]:30303
INFO [04-01|09:40:27] RLPx listener up                         self=enode://bac80bb4ca615dfbfbf8a738169ac72b47c7917a53941458981c4f09f3e1c0c98e940091acd12d66bff1927d4d44a26ee07814faf27c3f63b628ebc83e44e0ec@[::]:30303
INFO [04-01|09:40:27] IPC endpoint opened                      url=/home/mycroft/.ethereum/sweet-refrain/geth.ipc
```

Your private chain is running !

You can attach the console and retrieve the coins allocated in the genesis block (in the `alloc` section), using our private key:

```shell
$ geth --datadir .ethereum/sweet-refrain attach
Welcome to the Geth JavaScript console!

instance: Geth/v1.8.4-unstable-a095b84e/linux-amd64/go1.10.1
 modules: admin:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> personal.importRawKey('f0abab15e15b43826a746c89ceb740ad28b0fc683475d696f5c17d924cdd9294', '');
"0xc97ec1b4bf2b0106f951e113690b194289037d52"
> eth.accounts
["0xc97ec1b4bf2b0106f951e113690b194289037d52"]
> web3.fromWei(eth.getBalance(eth.accounts[0]))
1000000
> 
```

We can also mine new blocks. We need to set the Ether base before:

```javascript
> miner.setEtherbase('c97ec1b4bf2b0106f951e113690b194289037d52')
true
> miner.start(1)
null
> 
```

On first start, geth will generate the [DAG](https://ethereum.stackexchange.com/questions/1993/what-actually-is-a-dag) (which will take a few seconds) and then will mien blocks:

```
INFO [04-01|09:58:32] Successfully sealed new block            number=1 hash=65c64d…60aa82          
INFO [04-01|09:58:32] 🔨 mined potential block                  number=1 hash=65c64d…60aa82         
INFO [04-01|09:58:32] Commit new mining work                   number=2 txs=0 uncles=0 elapsed=123.754µs
```

You'll receive new coins in just a few seconds:

```javascript
> web3.fromWei(eth.getBalance('c97ec1b4bf2b0106f951e113690b194289037d52'), 'ether');
1000035
>
```

The reference of  APIs can be found on the [Javascript API](https://github.com/ethereum/wiki/wiki/JavaScript-API) and [Management APIs](https://github.com/ethereum/go-ethereum/wiki/Management-APIs) pages.

## Bonus: sending & receiving coins

Let's say now we want to send coins to another account. I'll create for this another address, bob's address, and I'll send him 5000 coins.

### Creating bob's account

Use `personal.newAccount` to create a new account.

```javascript
> personal.newAccount()
Passphrase: 
Repeat passphrase: 
"0x956dd1ec7e345ec5aed5d07938a239328e275b37"
> var bob = "0x956dd1ec7e345ec5aed5d07938a239328e275b37";
undefined
> web3.fromWei(eth.getBalance(bob))
0
```

Then, `eth.sendTransaction` to send coins.

```javascript
> var me = eth.accounts[0];
undefined
> web3.fromWei(eth.getBalance(me))
1000035
> var amount = web3.toWei(5000, "ether");
undefined
> web3.personal.unlockAccount(me, '', 600)
true
> eth.sendTransaction({from: me, to: bob, value: amount});
"0xddc5f0120fd3642eef7ef8fb7e76339f002b97160911a926193d0ccf2bc44f98"
```

At this time, the transaction is created and in the network, but miner is off. We need to mine at least a block for bob to be able to retrieve its coins.

```javascript
> miner.start(1)
null
INFO [04-01|10:19:23] Starting mining operation 
INFO [04-01|10:19:23] Commit new mining work                   number=8 txs=1 uncles=0 elapsed=454.946µs
INFO [04-01|10:19:25] Successfully sealed new block            number=8 hash=c43d9b…69d055
INFO [04-01|10:19:25] 🔨 mined potential block                  number=8 hash=c43d9b…69d055
INFO [04-01|10:19:25] Commit new mining work                   number=9 txs=0 uncles=0 elapsed=179.498µs
> miner.stop()
true
```

We mined a block, with one tx in it ! Let's check balances:

```javascript
> web3.fromWei(eth.getBalance(me))
995040
> web3.fromWei(eth.getBalance(bob))
5000
> 
```

Bob correctly got the coins!