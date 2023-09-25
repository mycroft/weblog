---
title:  "Sending a message in the blockchain"
date:   2018-03-11T08:33:17+0100
tags:
  - bitcoin
summary: "USing bitcoin scripting to send and store a message in the blockchain"
---

Recently, I was ask how easy it was to send a message in the blockchain. In Bitcoin, there are transactions, with input (coins that will be spent) & outputs (coins locking script), and in outputs, you can create [null data transactions][null-data-transaction] that are not spendable outputs (as they are pruned by `Bitcoin Core`), that embeds up to 83 bytes of data.

Using `bitcoin-cli`, creating such a [raw transaction][raw-transaction] is quite easy. Let's take a look at it.

You'll need to:

  * Find out an unspect transaction output to spent;
  * Build a new transaction, using that unspent output as input, and your target address to send the coins (removing the fees) to create a first output,
  * While creating this transaction, add a null data output as a second output;
  * Sign that transaction;
  * Send it to the network.

## Finding out an unspent output

There are a lot in the blockchain, but unfortunatelly, you'll need one you can spend. Let's find one using `listunspent`:

```shell
$ bitcoin-cli -testnet listunspent
[
  {
    "txid": "22d13746e7fccb4bff09271b54ea4b7cb1efe6371f3a1a0d511818e7ef7bbd04",
    "vout": 0,
    "address": "mgzhrk75WDYiiiNpUP3c6ixbSvFcPVGSzH",
    "account": "",
    "scriptPubKey": "76a9141037b8c9eb50aada3046511eb015176c4eb9de4d88ac",
    "amount": 0.16250000,
    "confirmations": 301,
    "spendable": true,
    "solvable": true,
    "safe": true
  }
]
```

This one will do it.

## Creating our transaction

Let's prepare our data. I want to write in the blockchain the following string: `Testing null data transaction for my weblog.`. It is a 44 (0x2c) bytes strings and its hex representation is:

```shell
$ echo "Testing null data transaction for my weblog." | xxd -pu -c 80
54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a
```

We will have to send `54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a` (the hex representation of the string).

The output I'll use can be use to spend 0.1625 coins. We need to take care of fees, and as our transaction will be ~140 bytes, I'll use a fee of `280 satoshis` (~2 satoshis/byte). I'll be able to recover `0.1625-(280/100000000) = .16249720`.

Now, to build our transaction, we need to use `createrawtransaction`:

```sh
$ export TXID=22d13746e7fccb4bff09271b54ea4b7cb1efe6371f3a1a0d511818e7ef7bbd04
$ export VOUT=0
$ export DATA=54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a
$ export AMOUNT=0.16249720
$ export ADDR=mgzhrk75WDYiiiNpUP3c6ixbSvFcPVGSzH
$ TX=$(bitcoin-cli -testnet createrawtransaction \
    "[{\"txid\":\"$TXID\", \"vout\":0}]" "{\"$ADDR\": $AMOUNT, \"data\": \"$DATA\"}"); echo $TX
020000000104bd7befe71818510d1a3a1f37e6efb17c4bea541b2709ff4bcbfce74637d1220000000000ffffffff0278f3f7
00000000001976a9141037b8c9eb50aada3046511eb015176c4eb9de4d88ac00000000000000002f6a2d54657374696e6720
6e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a00000000
```

We can take a look at our transaction using `decodetx`

```sh
bitcoin-cli -testnet decoderawtransaction $TX
{
  "txid": "268136ea83a8abdbb6f35de011df2a51d25a2aae17260c2d6171e1f2384c478c",
  "hash": "268136ea83a8abdbb6f35de011df2a51d25a2aae17260c2d6171e1f2384c478c",
  "version": 2,
  "size": 141,
  "vsize": 141,
  "locktime": 0,
  "vin": [
    {
      "txid": "22d13746e7fccb4bff09271b54ea4b7cb1efe6371f3a1a0d511818e7ef7bbd04",
      "vout": 0,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "sequence": 4294967295
    }
  ],
  "vout": [
    {
      "value": 0.16249720,
      "n": 0,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 1037b8c9eb50aada3046511eb015176c4eb9de4d OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a9141037b8c9eb50aada3046511eb015176c4eb9de4d88ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "mgzhrk75WDYiiiNpUP3c6ixbSvFcPVGSzH"
        ]
      }
    },
    {
      "value": 0.00000000,
      "n": 1,
      "scriptPubKey": {
        "asm": "OP_RETURN 54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a",
        "hex": "6a2d54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a",
        "type": "nulldata"
      }
    }
  ]
}
```

## Signing & sending

Next 2 steps are simpler. Signing will just take as an input the raw transaction, creating the signed transaction with the `signrawtransaction` rpc call. And finally, last step will be to inject the transaction in the network.

```sh
$ bitcoin-cli -testnet signrawtransaction $TX
{
  "hex": "020000000104bd7befe71818510d1a3a1f37e6efb17c4bea541b2709ff4bcbfce74637d122000000008a47304402201a8c3589
  440d3253bf1cd6162c68208ece41fe42b3d017a96cfa91993edab40702206749ec31f903247574f38772c79c8d0e99b4299912b958ac70
  0468b95efdc2c40141043f8008c64233bba3bbdfa300b11770c09f17ecfe49bc9a5d0260963136a7a0a9c28d7744a20008c026ce41825a
  f2d9007c288cbdf0af765eabdad85556de675cffffffff0278f3f700000000001976a9141037b8c9eb50aada3046511eb015176c4eb9de
  4d88ac00000000000000002f6a2d54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c
  6f672e0a00000000",
  "complete": true
}
$ export SIGNED_TX=020000000104bd7befe71818510d1a3a1f37e6efb17c4bea541b2709ff4bcb[...]
```

Last step is about to send our signed transaction, using `sendrawtransaction`:

```sh
$ bitcoin-cli -testnet sendrawtransaction $SIGNED_TX
63860881b1976b026916214cfd012a002ef273122c54fe437000e62fc2263f92
```

And we're done !

## Checking

Blocks explorer can give you information about your transaction, if it is confirmed or not, and show you the data send to the chain.

```sh
$ curl -s "https://api.blockcypher.com/v1/btc/test3/txs/63860881b1976b026916214cfd012a002ef273122c54fe437000e62fc2263f92?limit=50&includeHex=true" | \
    python3 -c "import sys, json; print(json.load(sys.stdin)['outputs'][1]['data_hex'])" | \
    xxd -r -p
Testing null data transaction for my weblog.
```

You can also use `gettransaction` and `decoderawtransaction` to check confirmations on the transaction and data sent:

```sh
$ bitcoin-cli -testnet gettransaction 63860881b1976b026916214cfd012a002ef273122c54fe437000e62fc2263f92
{
  "amount": 0.00000000,
  "fee": -0.00000280,
  "confirmations": 0,
  "trusted": true,
  "txid": "63860881b1976b026916214cfd012a002ef273122c54fe437000e62fc2263f92",
[...]
  "hex": "020000000104bd7befe71818510d1a3a1f37e6efb17c4bea541b2709ff4bcbfce74637d122000000008a47304402201a8c3589
  440d3253bf1cd6162c68208ece41fe42b3d017a96cfa91993edab40702206749ec31f903247574f38772c79c8d0e99b4299912b958ac70
  0468b95efdc2c40141043f8008c64233bba3bbdfa300b11770c09f17ecfe49bc9a5d0260963136a7a0a9c28d7744a20008c026ce41825a
  f2d9007c288cbdf0af765eabdad85556de675cffffffff0278f3f700000000001976a9141037b8c9eb50aada3046511eb015176c4eb9de
  4d88ac00000000000000002f6a2d54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c
  6f672e0a00000000"
}

$ bitcoin-cli -testnet decoderawtransaction 020000000104bd7befe718185[...]f672e0a00000000
{
  "txid": "63860881b1976b026916214cfd012a002ef273122c54fe437000e62fc2263f92",
  "hash": "63860881b1976b026916214cfd012a002ef273122c54fe437000e62fc2263f92",
[...]
    {
      "value": 0.00000000,
      "n": 1,
      "scriptPubKey": {
        "asm": "OP_RETURN 54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a",
        "hex": "6a2d54657374696e67206e756c6c2064617461207472616e73616374696f6e20666f72206d79207765626c6f672e0a",
        "type": "nulldata"
      }
    }
  ]
}
```

In a next article, I hope to redo the same thing, but programmatically using python or golang!

[raw-transaction]: https://en.bitcoin.it/wiki/Raw_Transactions
[null-data-transaction]: https://bitcoin.org/en/developer-guide#term-null-data
[push-opcode]: https://en.bitcoin.it/wiki/Script#Constants