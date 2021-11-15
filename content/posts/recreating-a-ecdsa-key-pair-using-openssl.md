---
title:  "Recreating a ecdsa key pair using Openssl"
date:   2018-03-24T16:05:02+0100
categories:
  - openssl
summary: "Quick tip to how generate a ecdsa key pair using command line openssl tools"
---

In one of my first posts, I described how to [extract a freshly created ecdsa private key to create a Bitcoin address][openssl-extract], and I was more recently asked [a way to reimport in a PEM or DER file a private key][openssl-import], and I kinda found it.

The procedure is to simply create a first DER file with a private key, replace the private by ours, and then regenerate the public key associated.

First, generate a first key in a DER file. The `-name secp256k1` param is mandatory here because we'll import the key, not the EC params, which are also mandatory to have a valid public key in our context.

```sh
$ openssl ecparam -name secp256k1 -genkey -noout -outform DER | \
     openssl ec -inform DER -no_public -outform DER -out template.der
read EC key
writing EC key
```

Then, with our private key in a hexadecimal format, create a new DER key with this key, and we check its ASN.1 structure and its validity.

```sh
$ head -c 7 template.der > private.key
$ echo a114dad00000000000faced00000000000bad000000000decaf00c02bad02bad | xxd -r -p >> private.key
$ tail -c +40 template.der >> private.key
$ ll private.key 
-rw-rw-r--. 1 mycroft mycroft 48 Mar 24 16:22 private.key
$ openssl asn1parse -inform DER < private.key
    0:d=0  hl=2 l=  46 cons: SEQUENCE          
    2:d=1  hl=2 l=   1 prim: INTEGER           :01
    5:d=1  hl=2 l=  32 prim: OCTET STRING      [HEX DUMP]:A114DAD00000000000FACED00000000000BAD000000000DECAF00C02BAD02BAD
   39:d=1  hl=2 l=   7 cons: cont [ 0 ]        
   41:d=2  hl=2 l=   5 prim: OBJECT            :secp256k1
$ openssl ec -check -inform DER < private.key 
read EC key
EC Key valid.
writing EC key
-----BEGIN EC PRIVATE KEY-----
MC4CAQEEIKEU2tAAAAAAAPrO0AAAAAAAutAAAAAA3srwDAK60CutoAcGBSuBBAAK
-----END EC PRIVATE KEY-----
$ 
```

Well, everything so far is OK ! Next step is to have Openssl to recreate a public key associated, and we'll be good to go.

```sh
$ openssl ec -inform DER -text -noout < private.key 
read EC key
Private-Key: (256 bit)
priv:
    a1:14:da:d0:00:00:00:00:00:fa:ce:d0:00:00:00:
    00:00:ba:d0:00:00:00:00:de:ca:f0:0c:02:ba:d0:
    2b:ad
pub:
    04:9f:7d:cb:14:14:21:77:d7:b9:48:78:c4:59:b6:
    3a:16:f4:12:80:84:49:b7:8f:a1:7b:e6:4c:d3:7f:
    ed:57:a6:42:12:07:e6:ca:95:e0:c5:15:c3:5f:d5:
    8c:af:ac:a8:b0:e7:d6:07:a3:3a:2c:5c:b1:6a:de:
    28:af:83:15:f7
ASN1 OID: secp256k1

$ openssl ec -inform DER -pubout < private.key 
read EC key
writing EC key
-----BEGIN PUBLIC KEY-----
MFYwEAYHKoZIzj0CAQYFK4EEAAoDQgAEn33LFBQhd9e5SHjEWbY6FvQSgIRJt4+h
e+ZM03/tV6ZCEgfmypXgxRXDX9WMr6yosOfWB6M6LFyxat4or4MV9w==
-----END PUBLIC KEY-----
```

This is it!

[openssl-extract]: https://mkz.me/gazette/bitcoin/2018/03/01/understanding-bitcoin-addresses.html
[openssl-import]: https://bitcointalk.org/index.php?topic=3180652