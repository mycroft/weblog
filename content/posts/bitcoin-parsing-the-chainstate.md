---
title:  "Bitcoin: Parsing the chainstate"
date:   2018-04-27T15:11:55+0200
tags:
  - bitcoin
summary: "How to track the spendable transaction in Bitcoin by exploring the chainstate database"
---

If you get a look at the files stored by the Bitcoin Core client, you'll see the big blocks directory, but another one caught my attention recently: The `chainstate` directory.

```bash
$ du -sh .bitcoin/*
[...]
4.0K    .bitcoin/bitcoin.conf
176G    .bitcoin/blocks
2.8G    .bitcoin/chainstate
[...]
$ 
```

In this directory, there is a leveldb database with a all currently unspent transactions outputs. In `Bitcoin`, the unspent transactions outputs (also called UTXOs) are one of the most important things in the protocol, as they are the only coins that can be spent. And this database is really smaller that the blockchain itself.

I've recently developed a tool that opens this database, and will parse all records to compute all balances for all addresses. This tool, that I called [chainstate](https://github.com/mycroft/chainstate), will in less than a few minutes take each records one by one, unobfuscate it (The `chainstate` database is obfuscated - simply by a xor, with a key which is inside the directory too), parse and understand the transaction format (p2pkh, p2sh, p2wpkh, p2wsh) to know what type of address we are dealing with, and output it.

The decoding is possible by reusing the bitcoin's [DecompressScript](https://github.com/bitcoin/bitcoin/blob/master/src/compressor.cpp#L96) function:

```cpp
        switch(script_type) {
            case 0x00: // P2PKH
                addr = get_addr(current_prefix.pubkey_prefix, value);
                cout << "DUP HASH160 " << value.size() << " " << addr << " EQUALVERIFY CHECKSIG" << endl; 
                break;

            case 0x01: // P2SH
                addr = get_addr(current_prefix.script_prefix, value);
                cout << "HASH160 " << value.size() << " " << addr << " EQUAL" << endl;
                break;

            case 0x02: // P2PK
            case 0x03:
                addr = get_addr(current_prefix.pubkey_prefix, str_to_ripesha(old_value));
                cout << "PUSHDATA(33) " << addr << " CHECKSIG" << endl;
                break;

            case 0x04: // P2PK
            case 0x05:
                memset(pub, 0, PUBLIC_KEY_SIZE);
                pubkey_decompress(script_type, value.c_str(), (unsigned char*) &pub, &publen);
                addr = get_addr(current_prefix.pubkey_prefix, str_to_ripesha(string((const char*)pub, PUBLIC_KEY_SIZE)));
                cout << "PUSHDATA(65) " << addr << " CHECKSIG" << endl;
                break;

            case 0x1c: // P2WPKH / P2WSH
            case 0x28:
                addr = rebuild_bech32(value);
                cout << "P2WSH "<< addr << endl;
                break;
        }
```

As the same code is used by the `Litecoin` and the `dashcore` clients, the same software is also supporting them too!

To use, we need to copy the chainstate binary (mandatory as it will overwrite some pointers in it, and Bitcoin Core would have to reindex its stuff, which is really slow) and well, wait:

```bash
$ cp -Rp ~/.bitcoin/chainstate state
$ ./chainstate >/tmp/cs.output 2>/tmp/cs.errors

$ head /tmp/cs.output
last block: 0000000000000000004e0f5635ad8b2e58ebd0a4f02c68c604d1b5697425ce72
eacfdcd42b27112ab6c8b435abec20181d05b0ba5d4f1829c002cc3ef0000000;1NwjXC31Enh5aqGHQbCtev9B7Rhk4knuEJ;1838
0118dd986e59473732239d39cb3b8890bf32677719dd8933b05f6614f4020000;32i3fvUTZkq2zeHBuosYDkiSCyMDhP62eo;132000
033e83e3204b0cc28724e147f6fd140529b2537249f9c61c9de9972750030000;1KaPHfvVWNZADup3Yc26SfVdkTDvvHySVX;65279
a53421b937be7bfe89ef6cc3f4124706b560af393b527e3e3d9d0c285b050000;1Lcd4mL7Zt53QTyR4wFJSksuyxCtfpTtws;2789

$ wc -l /tmp/cs.output /tmp/cs.errors
  59516004 /tmp/cs.output
    409643 /tmp/cs.errors
  59925647 total

$ 
```

