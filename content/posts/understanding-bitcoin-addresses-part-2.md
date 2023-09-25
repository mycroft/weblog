---
title:  "Understanding bitcoin addresses (part 2)"
date:   2018-03-08T07:14:09+0100
tags:
- bitcoin
summary: "Decrypting the Bitcoin addresses format (Part 2)"
---

This article is the following of [Understanding bitcoin addresses (part 1)][part-1]. We took a brief look in this article how to create with `openssl` a private/public ECDSA secp256k1 key and how to create from this a `Bitcoin address` and its private key in `Wallet Import Format (WIF)`.

There is still a lot to talk about `Bitcoin addresses`. This article will cover:

  * Uncompressed and compressed addresses;
  * Script ([P2SH] - [bip13]) addresses;
  * Newer [bech32][bech32] ([bip173]) addresses.

## Uncompressed/compressed addresses

### Public address

We saw in [part-1] that the size of an uncompressed key was 65 bytes, starting by 0x04. A single public key can also have a compressed representation. Its size will be 33 bytes, and will start by either 0x02 or 0x03.

In fact, a public key is composed by a couple of integers, x & y, and in its uncompressed form, the public key is simple _[0x04] [X value] [Y value]_. In its compressed form, the public key is _[0x02 or 0x03 prefix] [X value]_, because for an X value, the Y value can have only 2 values, an odd one and an even one. Then, the prefix will be 0x02 if Y is even, and 0x03 if it is odd.

Knowing this, the address generation script is very simple to update. We have only the `get_public_key` function to update:

```python
def get_public_key(privkey, use_compression = False):
    sk = ecdsa.SigningKey.from_string(privkey, curve=ecdsa.SECP256k1)
    vk = sk.verifying_key

    if use_compression:
        p = vk.pubkey.point
        order = sk.curve.generator.order()

        x_str = ecdsa.util.number_to_string(p.x(), order)
        prefix = bytes(chr((p.y() & 0x01) + 2), 'ascii')

        return (prefix + x_str)

    # Uncompressed
    return (b'\04' + sk.verifying_key.to_string())
```

The complete script is still available on my [repository][github-create-addr].

```sh
$ COMPRESS=1 python create-addr.py 4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33
Public key: 02ff3a56b2acbfdee1991c637fde89f0419833a1dc2e6f28dbe8ee64a3980b7c5d
Public address: 1LQWpoxq7hAwGu8AXfHG7hWdN9bt9dRnzp

# Verifying with bitcoin-tool:
$ ./bitcoin-tool --input 4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33 \
                 --input-type private-key --input-format hex --output-type address \
                 --output-format base58check --network bitcoin --public-key-compression compressed
1LQWpoxq7hAwGu8AXfHG7hWdN9bt9dRnzp
```

### Private address

Again, the change is light for private addresses. There is no compress form for private key, so the only change is to inform that the address is compressed, by adding a compression flag 0x01 after the private key. The format will then be _[prefix] [32 bytes private-key] [compression flag: 0x01] [checksum]_. With this form, the private key prefix will change from 5 to L or K, as documented on [List of address prefixes][list-address-prefix].

The change in our python code is a one-liner:

```python
if use_compression:
    extprivkey += b'\x01'

# Creating checksum...
```

Let's perform a quick check:

```sh
$ COMPRESS=1 python create-addr.py 4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33
Public key: 02ff3a56b2acbfdee1991c637fde89f0419833a1dc2e6f28dbe8ee64a3980b7c5d
Public address: 1LQWpoxq7hAwGu8AXfHG7hWdN9bt9dRnzp
Private key: 804e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa3301ffcc81a7
Private WIF address: KyrGe2egaPrAwppagzXQnQeVF13z2BpCtrShRNhcVBkLLDLtgch4

# And with bitcoin-tool:
$ ./bitcoin-tool --input 4e7b7fb13697e6940c08d0bd9adefdea127047797bcdea85bb25d8bdd101aa33 \
                 --input-type private-key --input-format hex --output-type private-key-wif \
                 --output-format base58check --network bitcoin --public-key-compression compressed
KyrGe2egaPrAwppagzXQnQeVF13z2BpCtrShRNhcVBkLLDLtgch4
```

We are good ! Last step: just add this key to `Bitcoin Core` and check we are still good.

```sh
$ bitcoin-cli importprivkey 'KyrGe2egaPrAwppagzXQnQeVF13z2BpCtrShRNhcVBkLLDLtgch4' 'my-compressed-key' false
$ bitcoin-cli getaddressesbyaccount 'my-compressed-key'
[
  "1LQWpoxq7hAwGu8AXfHG7hWdN9bt9dRnzp"
]
$ bitcoin-cli dumpprivkey '1LQWpoxq7hAwGu8AXfHG7hWdN9bt9dRnzp'
KyrGe2egaPrAwppagzXQnQeVF13z2BpCtrShRNhcVBkLLDLtgch4
```

Addresses are correct. Don't forget the `false` flag or you'll have to wait 30 minutes for `bitcoind` to rescan the whole chain!

## Script addresses

Script addresses, or [P2SH] addresses were introduced in [bip-16][bip16] to allow transactions to be sent to a `script hash` instead of a `public key hash`. The main difference is that we have to provide a script matching the script hash to spend an output, and then, this provides more security that just using a public key/private key, as we can use more than one key to protect our transaction using OP_CHECKMULTISIG, for example. These addresses are starting with the integer 3 (ex: `3G734WzCrphZxN7afnrbwunZjV8MBqWUUV`) and their prefix is 0x05, as set in [Bitcoin Core][bitcoin-chainsparams]:

```c++
        base58Prefixes[PUBKEY_ADDRESS] = std::vector<unsigned char>(1,0);
        base58Prefixes[SCRIPT_ADDRESS] = std::vector<unsigned char>(1,5);
        base58Prefixes[SECRET_KEY] =     std::vector<unsigned char>(1,128);
```

For information, values for testnet are:

```c++
        base58Prefixes[PUBKEY_ADDRESS] = std::vector<unsigned char>(1,111);
        base58Prefixes[SCRIPT_ADDRESS] = std::vector<unsigned char>(1,196);
        base58Prefixes[SECRET_KEY] =     std::vector<unsigned char>(1,239);
```

I found out a [nice article][multisig-transactions] to understand possibilities using P2SH address.

Creating a P2SH address is not that simple than previous ones. We need a script (that we'll use to create a hash), and its redeemscript (the script that will allow to unlock the transaction).

Let's use bitcoin-cli to create our address. We need at least a public key, so we will create a legacy address:

```sh
$ bitcoin-cli -testnet getnewaddress '' legacy
n2YzkhomdGtLx3x2SZkxwrS14kD9UiMWvR

$ bitcoin-cli -testnet validateaddress n2YzkhomdGtLx3x2SZkxwrS14kD9UiMWvR
{
  "isvalid": true,
  "address": "n2YzkhomdGtLx3x2SZkxwrS14kD9UiMWvR",
  "scriptPubKey": "76a914e6bd86c622cbda7c348e8819021c3b7fa3dd6a7c88ac",
  "ismine": true,
  "iswatchonly": false,
  "isscript": false,
  "iswitness": false,
  "pubkey": "03c09c6ebdebfe02bbbbacd77687e17e5361d07b54efb392e53829edf4f46bd4ee",
  "iscompressed": true,
  "account": "",
  "timestamp": 1509793090,
  "hdkeypath": "m/0'/0'/7'",
  "hdmasterkeyid": "f29391c2b992c59995b7bf978d572d317650cd16"
}
```

... and then the script address:

```sh
$ bitcoin-cli -testnet createmultisig 1 '["03c09c6ebdebfe02bbbbacd77687e17e5361d07b54efb392e53829edf4f46bd4ee"]'
{
  "address": "2NGAz9ShNSXPWBtg9DPvCjxsx6XEr9pnZHs",
  "redeemScript": "512103c09c6ebdebfe02bbbbacd77687e17e5361d07b54efb392e53829edf4f46bd4ee51ae"
}
```

We case easily parse the redeemscript generated `512103c09c6ebdebfe02bbbbacd77687e17e5361d07b54efb392e53829edf4f46bd4ee51ae`:

```sh
    51 21 03c09c6ebdebfe02bbbbacd77687e17e5361d07b54efb392e53829edf4f46bd4ee 51 ae
    <OP_TRUE> <push 0x21 bytes> <0x21 bytes of public key> <OP_TRUE> <OP_CHECKMULTISIG>
```

And we're done, as our address is `2NGAz9ShNSXPWBtg9DPvCjxsx6XEr9pnZHs` ! We can now use that address to retrieve bitcoins, and we'll have to use the redeemscript (and private key of embedded public key) to unlock our coins.

Let's update our python code:

```python
# Create script address from this pubkey
# \x51 is OP_TRUE
# \xae is OP_MULTICHECKSIG
public_key = get_public_key(privkey, True)
script = b'\x51' + bytes(chr(len(public_key)), 'ascii') + public_key + b'\x51' + b'\xae'

addr_sha256 = sha256(script)
addr_ripemd160 = ripemd160(addr_sha256)

extended_ripemd160 = prefix_script + addr_ripemd160

check = sha256(extended_ripemd160)
check = sha256(check)

addr = extended_ripemd160 + check[:4]
addr = base58.b58encode(addr)

print("Redeem script: %s" % script.hex())
print("Public script address: %s" % addr)
```

```sh
$ python create-addr.py 34c8fa74bec5bc1bea9c2789cf980d2b6b37d556e7c27127c84bbd7022135a09
...
Redeem script: 512103c09c6ebdebfe02bbbbacd77687e17e5361d07b54efb392e53829edf4f46bd4ee51ae
Public script address: 2NGAz9ShNSXPWBtg9DPvCjxsx6XEr9pnZHs
```

If you need to spend those, you should refer to [this developer example][p2sh-multisig]. Also, you can import your private key, and coins will be usable.

## bech32 addresses

Bech32 was introduced by [bip173] for segwit transactions. It uses a new hashing format and is supported since `Bitcoin Core 0.16`.

Like p2sh addresses above, we will use `bitcoin-cli` to create an address.

```sh
$ bitcoin-cli -testnet getnewaddress '' bech32
tb1qqfx0qelssry58amg9cvnw7m4f8d2jxcynw32ur

$ bitcoin-cli -testnet dumpprivkey tb1qqfx0qelssry58amg9cvnw7m4f8d2jxcynw32ur
cVcoCBFaRipj7J1uCgN5JCqRhoo6XajCE9oeYA1qTxyDWemroVC5
```

The format is simple to understand. It is composed by a prefix (hrp, human readable part, `bc` for bitcoin's mainnet, `tb` for bitcoin's testnet), a program version and a witness program. for a simple p2wpkh (pay to witness public key hash), the witness program is simply `ripemd160(sha160(compressed public key))`. In case of more secure p2wsh address, the witness program will be the `sha256-hashed` (and not ripemd160(sha256())-hashed!) `p2sh script` (or redeem script), as described in [bip141][bip141-witness-program].

We will need a bech32 implementation. There is a lot of samples for multiple languages in this [repository][sipa-bech32]. Generating `bech32 address` will be simple as just calling the encode function:

```python
# We have downloaded segwit_addr.py script in the current directory.
sys.path.append(os.path.abspath("."))
import segwit_addr

# ... We already have a pubkey defined ...

# Perform sha256 on publickey
addr_sha256 = sha256(pubkey)

# Then, ripemd160 on result
addr_ripemd160 = ripemd160(addr_sha256)

# tb for testnet, bc for mainnet
bc32 = segwit_addr.encode("tb", 0, addr_ripemd160)
print("bech32 (p2wpkh): %s" % bc32)

public_key = get_public_key(privkey, True)
script = b'\x51' + bytes(chr(len(public_key)), 'ascii') + public_key + b'\x51' + b'\xae'

addr_sha256 = sha256(script)

bc32 = segwit_addr.encode("tb", 0, addr_sha256)
print("bech32 (p2wsh): %s" % bc32)
```

When executing:

```sh
$ python create-addr.py cVcoCBFaRipj7J1uCgN5JCqRhoo6XajCE9oeYA1qTxyDWemroVC5
[...]
bech32 (p2wpkh): tb1qqfx0qelssry58amg9cvnw7m4f8d2jxcynw32ur
bech32 (p2wsh): tb1qzhh2d3vjtdhhzpdv6lwufn5surfkgm0880cfqm5lqwa3xl54e0lsm8qhuu
```

## Other ways to generate private keys

As there are another methods to generate private keys that will need a new post to talk about:

  * [Mnemonic code for generating deterministic keys][bip39]: A way to create a key from human words;
  * [Hierarchical determninistic wallets][bip32]: How to generate a lot of public/private keys from a single seed.

[part-1]:       https://mkz.me/gazette/bitcoin/2018/03/01/understanding-bitcoin-addresses.html
[P2SH]:         https://en.bitcoin.it/wiki/Pay_to_script_hash
[bip13]:        https://github.com/bitcoin/bips/blob/master/bip-0013.mediawiki
[bip16]:        https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
[bip32]:        https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
[bip39]:        https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
[bip173]:       https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
[bech32]:       https://en.bitcoin.it/wiki/Bech32
[github-create-addr]: https://github.com/mycroft/gazette-samples/tree/master/bitcoin-addresses
[list-address-prefix]: https://en.bitcoin.it/wiki/List_of_address_prefixes
[bitcoin-chainsparams]: https://github.com/bitcoin/bitcoin/blob/master/src/chainparams.cpp#L140
[multisig-transactions]: http://www.soroushjp.com/2014/12/20/bitcoin-multisig-the-hard-way-understanding-raw-multisignature-bitcoin-transactions/
[scripts]:      https://en.bitcoin.it/wiki/Script
[be-testnet-ex1]: https://live.blockcypher.com/btc-testnet/tx/7417a79f7bcea57e046e5488d176345302e6c61452aac9c73acdd5d7b73481e8/
[p2sh-multisig]: https://bitcoin.org/en/developer-examples#p2sh-multisig
[bip141-witness-program]: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#witness-program
[sipa-bech32]:   https://github.com/sipa/bech32