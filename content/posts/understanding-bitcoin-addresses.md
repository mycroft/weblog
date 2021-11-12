---
title:  "Understanding bitcoin addresses (part 1)"
date:   2018-03-01 07:58:09 +0100
categories:
- bitcoin
summary: "Decrypting the Bitcoin addresses format"
---

My first post on this journal will be dedicated to `Bitcoin addresses`. As addresses are in the hearth of Bitcoin's protocol and softwares (and basically, allowing keeping a lot of coins), they changed a lot in the past decade, supporting multiple formats and with different tools to generate and manipulate them.

On this page, I'll try to write most information I've collected on those the last monthes.

## Creating a Bitcoin address

At the beginning, a Bitcoin address is simply a representation of a public key generated from a ECDSA [secp256k1] private key, and using a small hashing algorithm.

### Using Openssl to create a ECDSA secp256k1 private & public key

On this [Stackexchange post][stackexchange-post], there is method to create a private/public ECDSA key using `openssl` tool.

Firstly, let's create a private key in `PEM format`.

```sh
$ openssl ecparam -genkey -name secp256k1 -rand /dev/urandom -out private.key
```

Then, its related public key, in PEM format too.

```sh
$ openssl ec -in private.key -pubout -out public.pem
```

As those are PEM files, we need to extract private key bytes in order to use them to be able to import them in [Bitcoin core][bitcoin-core] client.

```sh
$ BTC_PRIVATE_KEY=$(openssl ec -in private.key -outform DER|tail -c +8|head -c 32|xxd -p -c 32)
$ echo $BTC_PRIVATE_KEY
4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33
```

If you need to understand `DER format`, you can analyze its structure by using `openssl asn1parse`:

```sh
$ openssl ec -in private.key -outform DER | openssl asn1parse -in - -inform DER
```

(Key is at the 8th byte because we have sequence information, then an integer before reaching our key. Some ASN.1 format knowledge is mandatory!)

Then, grab the public key bytes, but this is optional as you'll be able to get them again using private key when preparing your Bitcoin address.

```sh
$ BTC_PUBLIC_KEY=$(openssl ec -in private.key -pubout -outform DER|tail -c 65|xxd -p -c 65)
$ echo $BTC_PUBLIC_KEY
04ff3a56b2acbfdee1991c637fde89f0419833a1dc2e6f28dbe8ee64a3980b7c5d1fd133fcc43e508def2a630ef9719bcce213c489e9ec70d1d7a9fedb0f3dccc8
```

Those keys are not yet usable in Bitcoin, but if you are curious enough, you'll note that this public key & the [Bitcoin's genesis block one][bitcoin-genesis] are the same format: `65 bytes, starting by 0x04`.

## Getting our Bitcoin address

### Public address

There is still some work, using the public key we've just got, to have a ready to use Bitcoin address. The processus is described [right there][technical-background-v1-addresses], but let's take a deeper look.

#### Step 0: We have our ECDSA private key

In my examples, my private key is `4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33`, and it is a 32 byte key. So far, so good.

#### Step 1: We also have corresponding public key.

In its uncompressed format, our public key is 65 bytes long, and starting by 0x04. Public key also have compressed form, starting by either 0x02 or 0x03, and we will talk about it later.
My public key is `04ff3a56b2acbfdee1991c637fde89f0419833a1dc2e6f28dbe8ee64a3980b7c5d1fd133fcc43e508def2a630ef9719bcce213c489e9ec70d1d7a9fedb0f3dccc8`.

#### Step 2: We perform SHA-256 on our private key.

SHA-256 hash is `a701507ec74fd980a9ce5c1ecdb4622ec962e72ea6133ca6b0ec20d402a89b1a`.

#### Step 3: We now perform RIPEMD-160 on the last result.

RIPEMD-160 hash is `134fb77c56423e86aae9d4076d1f4b37bfb6c74c`.

#### Step 4: We need to add network prefix.

The network prefix must be added. This prefix is `0x00` for Bitcoin main network, but differs along different networks & testnets. Bitcoin testnet's is 0x6f, P2SH's prefix is `0x05`, etc. Other coins, such litecoin, have their own set of prefix too. This [page][list-address-prefix] has an exhaustive list of Bitcoin's address prefixes.

Then, our key is now `00134fb77c56423e86aae9d4076d1f4b37bfb6c74c`.

Let's store this somewhere, as we will be using it at the end. Next steps are about creating a checksum of this key for validation purposes.

#### Step 5: Perform a SHA-256 hash on the latest result

SHA-256 hash is `ec0b051096187b915374e38fe608121112762133c426a63c97a5de7bad3129dc`

#### Step 6: Again, perform a last SHA-256 hash

It is now `83562b05e49caefd2c3c200253d50616e1b46551ceb483f8ed7974cc45778e89`

#### Step 7: Appending first 4 bytes of last checksum to extended RIPEMD-160 key

From the last hash computed in `step 6`, we take the first 4 bytes, `83562b05`, and we append to the RIPEMD-160 hash we got at `step 4` (the one starting by the network prefix, `0x00`).

Our key is now `00134fb77c56423e86aae9d4076d1f4b37bfb6c74c83562b05`.

#### Final step: Encode this key using base58.

The key becomes `12m7LTjx5vcx23MeumZuNtAyG3qEcj2rn8`. This is our bitcoin address.

Let's take a deeper look with some real python code. Source code for this script is also available on [github][github-create-addr].

```python
#!/usr/bin/env python

import base58
import binascii
import ecdsa
import hashlib
import sys

if len(sys.argv) != 2:
    print("Usage: %s <private key in hex format>" % sys.argv[0])
    sys.exit(1)

"""
Using ecdsa primitives, retrieve public key from private key.
"""
def get_public_key(privkey):
    sk = ecdsa.SigningKey.from_string(privkey, curve=ecdsa.SECP256k1)
    vk = sk.verifying_key
    return (b'\04' + sk.verifying_key.to_string())

def sha256(content):
    m = hashlib.new('sha256')
    m.update(content)
    return m.digest()

def ripemd160(content):
    m = hashlib.new('ripemd160')
    m.update(content)
    return m.digest()

privkeyhex = sys.argv[1]
privkey = binascii.unhexlify(privkeyhex)
pubkey = get_public_key(privkey)

# Perform sha256 on publickey
addr_sha256 = sha256(pubkey)

# Then, ripemd160 on result
addr_ripemd160 = ripemd160(addr_sha256)

# And add network byte as a prefix (0x00 on bitcoin's mainnet)
extended_ripemd160 = b'\x00' + addr_ripemd160

# Let's compute hash check on this string, using sha256 twice:
check = sha256(extended_ripemd160)
check = sha256(check)

# Add the first 4 bytes of check at the end of address (extended_ripe160):
addr = extended_ripemd160 + check[:4]

# Perform base58 encode to get final address:
addr = base58.b58encode(addr)

print(addr)
```

Let's now run it:

```sh
$ ./create-addr.py 4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33
12m7LTjx5vcx23MeumZuNtAyG3qEcj2rn8
```

Here we go. We have our brand fresh new bitcoin address (in its 1st version format and uncompressed). There is other formats for this public key: its compressed form, its P2SH (starting by 3) form, and its bech32 (segwit) form. We'll talk about it later.

### Private address

Like public address, bitcoin is using its own format for importing/exporting private keys. It's called the [Wallet import format][wallet-import-format].

Steps to create a WIF from a raw private key are easy.

#### Step 0: We have our ECDSA private key (again)

My private key still is `4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33`.

#### Step 1: We add the main net prefix for private keys.

Like public keys, you've to add a prefix. This prefix is `0x80` for mainnet private keys, and `0xef` for testnet addresses.

Result is `804e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33`

#### Step 2: Preparing checksum (using SHA-256 twice)

Again, let's run SHA-256 twice on this:

First run `09dce5d2b69be93f0c858e76f0175cb41b58ccc2e28b9f8a533f1daf89cb096d`
Second run `202f67126771f411cd557e170325e46fec2386ab8cf8a744a49d95f3d5e96e6c`

And we note down the first 4 bytes of the last hash.

#### Step 3: Append 4 first bytes of checksum to prefixed key

My private key is now `804e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33202f6712`

#### Final step: Perform base58 encode:

My private key in its WIF format is `5JQrJuV2GGbhmbQ1qUzBpu2hPCnzKr85492k9EynCNU37deUgqo`.


Just add the following lines to the python script to compute this address:

```python
extprivkey = b'\x80' + privkey

check_privkey = sha256(extprivkey)
check_privkey = sha256(check_privkey)

extprivkey = extprivkey + check_privkey[:4]

privaddr = base58.b58encode(extprivkey)

print("Private WIF address: %s" % privaddr)
```

You can run it too:

```sh
$ ./create-addr.py 4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33
Public address: 12m7LTjx5vcx23MeumZuNtAyG3qEcj2rn8
Private WIF address: 5JQrJuV2GGbhmbQ1qUzBpu2hPCnzKr85492k9EynCNU37deUgqo
```

### Checking with Bitcoin Core client.

It is easy to check this private & public key couple: Just import the private key to `Bitcoin Core` using debug console or bitcoin-cli. In this sample, we need an up to date `Bitcoin Core` client.

```sh
$ /opt/bitcoin/bin/bitcoin-cli walletpassphrase "my wallet passphrase"
$ /opt/bitcoin/bin/bitcoin-cli importprivkey 5JQrJuV2GGbhmbQ1qUzBpu2hPCnzKr85492k9EynCNU37deUgqo "my-testing-key"
$ /opt/bitcoin/bin/bitcoin-cli getaddressesbyaccount "my-testing-key"
[
  "12m7LTjx5vcx23MeumZuNtAyG3qEcj2rn8"
]
$ /opt/bitcoin/bin/bitcoin-cli dumpprivkey 12m7LTjx5vcx23MeumZuNtAyG3qEcj2rn8
5JQrJuV2GGbhmbQ1qUzBpu2hPCnzKr85492k9EynCNU37deUgqo
```

As you can see, my private key was correctly imported in `Bitcoin core`, and I'm able to export its associated public address, and from this public address, my private key!

## In a nutshell

There is a great tool allowing to manipulate bitcoin addresses. It's [bitcoin-tool on github][bitcoin-tool], and I used it a lot during some of my investigation for debugging purpose. Unfortunately, it is not supporting all addresses types ([Pay to script hash][P2SH], [bech32]).

This tool allows creating for a private key everything we need, but it is less fun than creating our keys by ourselves!

```sh
$ ./bitcoin-tool --network bitcoin --public-key-compression uncompressed \
                 --input 4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33 \
                 --input-type private-key --input-format hex --output-type all
address.base58:1Gc3y1NC389SvmsrPnjYicvGxZT1
address.base58check:12m7LTjx5vcx23MeumZuNtAyG3qEcj2rn8
public-key-ripemd160.hex:134fb77c56423e86aae9d4076d1f4b37bfb6c74c
public-key-ripemd160.base58:Gc3y1NC389SvmsrPnjYicvGxZT1
public-key-ripemd160.base58check:2m7LTjx5vcx23MeumZuNtAyG3qEcQWUnE
public-key-sha256.hex:a701507ec74fd980a9ce5c1ecdb4622ec962e72ea6133ca6b0ec20d402a89b1a
public-key-sha256.base58:CEvFPLR67K5w9hasgguqXLeCwSp6QbzK5Z6QeH4qQmku
public-key-sha256.base58check:2GYvUfUWbxfMAeBBNSb95tXVf5pG3ravsUsb1TyRyH3nULQzA7
public-key.hex:04ff3a56b2acbfdee1991c637fde89f0419833a1dc2e6f28dbe8ee64a3980b7c5d1fd133fcc43e508def2a630ef9719bcce213c489e9ec70d1d7a9fedb0f3dccc8
public-key.base58:SaQCjQrh5LxRhJt2y73knCW4QDRRM3NtCVY28zLf9JmmcsJscSLy1uJ3qDAPy3TS9AsmCv1o4GAFSMxS5kJLJ6g7
public-key.base58check:3tMr7x4xAQEFLCzGnRT8unap35ZAh8squKYT4oZuSHPXJ6FHzrNApBfHSB4XgRqh944tLMLta5PbYpqTbPkaLTLE5CmEFp
private-key-wif.hex:804e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33
private-key-wif.base58:f7fuTteYWo2mxZvRsfFXjyr9cxXAUxSej5GtceXszKJPt
private-key-wif.base58check:5JQrJuV2GGbhmbQ1qUzBpu2hPCnzKr85492k9EynCNU37deUgqo
private-key.hex:4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33
private-key.base58:6HN2MgKjAzPbZNCQanooDbUryJo7bocdAknzUdDkDNta
private-key.base58check:bZjE2wBp2iWb1joHYHjrHaAby57iFzLKxxv4w2xVcPx66KR9j
```


## What's next ?

There is still a lot to say about Bitcoin's addresses:
* compressed keys;
* P2SH addresses;
* bech32 addresses;
* deterministic keys.

I'll write about these in a next post!

[stackexchange-post]: https://bitcoin.stackexchange.com/questions/59644/how-do-these-openssl-commands-create-a-bitcoin-private-key-from-a-ecdsa-keypair
[bitcoin-core]: https://bitcoin.org/en/download
[secp256k1]:    http://www.secg.org/sec2-v2.pdf
[bitcoin-genesis]: https://github.com/bitcoin/bitcoin/blob/0.16/src/chainparams.cpp#L52
[technical-background-v1-addresses]: https://en.bitcoin.it/wiki/Technical_background_of_version_1_Bitcoin_addresses
[wallet-import-format]: https://en.bitcoin.it/wiki/Wallet_import_format
[bitcoin-tool]: https://github.com/matja/bitcoin-tool
[P2SH]:         https://en.bitcoin.it/wiki/Pay_to_script_hash
[bech32]:       https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
[list-address-prefix]: https://en.bitcoin.it/wiki/List_of_address_prefixes
[github-create-addr]: https://github.com/mycroft/gazette-samples/tree/master/bitcoin-addresses