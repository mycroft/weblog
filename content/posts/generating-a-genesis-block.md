---
title:  "Generating a genesis block"
date:   2018-03-18T13:34:56+0100
tags:
  - bitcoin
summary: "How to generate a Bitcoin Genesis block"
---

`Bitcoin` is a blockchain, and its chain has an original, the [Genesis block][genesis-block]. The whole chain depends on that block.

The Genesir block is a regular block, and is composed of the same structure than any other [block]. It has a single [transaction] with an output that is by design not spendable.

```go
type Block struct {
    VersionNumber uint32
    PreviousHash  []byte
    Hash          []byte
    MerkleRoot    []byte
    Timestamp     uint32
    Bits          uint32
    Nonce         uint32
    TxCount       int
    Txs           []*Transaction
}
```

In the `Bitcoin` blockchain, the Genesis block's [coinbase] (input of the generating transaction, not used by the protocol) contains a single sentence, to date the block creation. This sentence is `The Times 03/Jan/2009 Chancellor on brink of second bailout for banks`. Litecoin's is `NY Times 05/Oct/2011 Steve Jobs, Apple’s Visionary, Dies at 56`.

We can easily retrieve the complete block using bitcoin-cli (or json-api rpc):

```sh
$ bitcoin-cli getblockhash 0
000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f

$ bitcoin-cli getblock 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f false
0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e677
68f617fc81bc3888a51323a9fb8aa4b1e5e4a29ab5f49ffff001d1dac2b7c01010000000100000000000000000000000000
00000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f3
2303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e
6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb
649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac00000000
```

The structure is the following:

  * 01000000: version
  * 0000000000000000000000000000000000000000000000000000000000000000: previous block hash
  * 3ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a: merkle root
  * 29ab5f49: timestamp
  * ffff001d: difficulty
  * 1dac2b7c: nonce
  * 01: num of transactions
  * -- transaction 1
    * 01000000: version
    * 01: num of inputs
    * 0000000000000000000000000000000000000000000000000000000000000000: tx hash from the output
    * ffffffff: tx output number
    * 4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73: input script: contains the genesis' sentence
    * ffffffff: sequence
    * 01: num of outputs
    * 00f2052a01000000: value
    * 434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f1deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac: output script
    * 00000000 locktime
  * -- end of transaction 1

When you want to create your own chain (and building your own altcoin), you have to regenerate a new genesis block, and a quick way but ugly is to patch your wallet source, then launch it in order to compute (mine) the initial transaction & block, then report it in the source, and recompile the wallet. This can be a difficult step if you don't know well C++ or how chains work.

There is a few tools to do this, but I wanted to do my own in golang, and the result is a new project called [generate-genesis]. It is a simple CLI command that takes all needed parameters for a new Genesis block, and will iterate nonce & timestamp until a new valid hash is found.

```sh
$ ./generate-genesis -h
Usage of ./generate-genesis:
  -algo string
        Algo to use: sha256, scrypt (default "sha256")
  -bits string
        Bits (default "1d00ffff")
  -coins uint
        Number of coins (default 5000000000)
  -nonce uint
        Nonce value (default 2083236893)
  -psz string
        pszTimestamp (default "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks")
  -pubkey string
        Pubkey (required) (default "04678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5f")
  -timestamp uint
        Timestamp to use (default 1231006505)
```

An interesting part of the code is the block hash validity check, which differs if you are computing a `sha256` hash (for bitcoin) or a `scrypt` hash. There another hashes, but I didn't studied them yet. We need in this code to compute from the compact bits the [target difficulty][target-difficulty], then compute the sha256/scrypt/... hash, compare it against the target difficulty, and if the hash (which is considered as a big integer) is smaller than target, we consider the hash is valid. If not, we increment nonce and restart the process.

```go
    target := ComputeTarget(blk.Bits)

    for {
        switch params.Algo {
        case "sha256":
            hash = ComputeSha256(ComputeSha256(blk.Serialize()))
        case "scrypt":
            hash = ComputeScrypt(blk.Serialize())
        }

        current.SetBytes(Reverse(hash))
        if 1 == target.Cmp(&current) {
            break
        }

        blk.Nonce ++
        if blk.Nonce == 0 {
            blk.Timestamp ++
        }
    }
```

By default, the [Generate Genesis][generate-genesis] tool will compute & a valid nonce for a new block for the bitcoin chain:

```sh
$ ./generate-genesis 
Ctrl Hash:      0x000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
Target:         0x00000000ffff0000000000000000000000000000000000000000000000000000
Blk Hash:       0x000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
Mkl Hash:       0x4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
Nonce:          2083236893
Timestamp:      1231006505
Pubkey:         04678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5f
Coins:          5000000000
Psz:            'The Times 03/Jan/2009 Chancellor on brink of second bailout for banks'
```

You can change nonce value, to make sure it didn't cheat, and it will recompute it:

```sh
$ time ./generate-genesis -nonce 2080000000|grep Nonce
Nonce:          2083236893

real    0m4.757s
user    0m4.934s
sys     0m0.072s
```

You can also compute the same thing for litecoin:

```sh
$ ./generate-genesis -algo scrypt -psz "NY Times 05/Oct/2011 Steve Jobs, Apple’s Visionary, Dies at 56" -coins 5000000000 -pubkey 040184710fa689ad5023690c80f3a49c8f13f8d45b8c857fbcbc8bc4a8e4d3eb4b10f4d4604fa08dce601aaf0f470216fe1b51850b4acf21b179c45070ac7b03a9 -timestamp 1317972665 -nonce 2084524493 -bits 1e0ffff0
Ctrl Hash:      0x0000050c34a64b415b6b15b37f2216634b5b1669cb9a2e38d76f7213b0671e00
Target:         0x00000ffff0000000000000000000000000000000000000000000000000000000
Blk Hash:       0x12a765e31ffd4059bada1e25190f6e98c99d9714d334efa41a195a7e7e04bfe2
Mkl Hash:       0x97ddfbbae6be97fd6cdf3e7ca13232a3afff2353e29badfab7f73011edd4ced9
Nonce:          2084524493
Timestamp:      1317972665
Pubkey:         040184710fa689ad5023690c80f3a49c8f13f8d45b8c857fbcbc8bc4a8e4d3eb4b10f4d4604fa08dce601aaf0f470216fe1b51850b4acf21b179c45070ac7b03a9
Coins:          5000000000
Psz:            'NY Times 05/Oct/2011 Steve Jobs, Apple’s Visionary, Dies at 56'
```

The source code is available on my [private gitlab][generate-genesis] for registered users.

Update 2018/03/19: [generate-genesis][generate-genesis] now works with the X11 proof of work!

[genesis-block]:     https://en.bitcoin.it/wiki/Genesis_block
[block]:             https://en.bitcoin.it/wiki/Block
[transaction]:       https://en.bitcoin.it/wiki/Transaction
[coinbase]:          https://en.bitcoin.it/wiki/Coinbases
[generate-genesis]:  https://gitlab.mkz.me/mycroft/generate-genesis
[target-difficulty]: https://en.bitcoin.it/wiki/Difficulty
