---
title: "OpenSSL: First steps with Hashing and crypt functions"
date: 2010-05-07
summary: "Basic play with OpenSSL hashing & crypt functions"
tags:
  - openssl
---

As I'm starting to work with [OpenSSL](https://www.openssl.org/) to rewrite my [password wallet](http://github.com/mycroft/mnkPasswordKeeper), which uses [ECB mode](http://en.wikipedia.org/wiki/Block_cipher_modes_of_operation) to store data, I wanted to write down a few notes about how to use it.

My first step was to know how OpenSSL worked. Here a few useful snippets to do basic tasks:

sha256 on a string
------------------

```c
    SHA256_CTX sha256;
    SHA256_Init(&sha256);
    SHA256_Update(&sha256, pass, strlen(pass));
    SHA256_Final(key, &sha256);
```

AES 256, CBC crypt
------------------

```c
    EVP_CIPHER_CTX ctx;
    const EVP_CIPHER *cipher_type
    unsigned char *iv, *key;

    /* Don't forget to initialize memory for iv and key */
    /* Don't forget to set key as well */

    EVP_CIPHER_CTX_init(&ctx);

    cipher_type = EVP_aes_256_cbc();
    EVP_EncryptInit_ex(&ctx, cipher_type, NULL, NULL, NULL);
    RAND_bytes(iv, EVP_CIPHER_CTX_iv_length(&ctx));

    EVP_EncryptInit_ex(&ctx, NULL, NULL, key, iv);
    // 'in' contains message to encrypt, 'in_len' its length.
    EVP_EncryptUpdate(ctx, out, &out_len, in, in_len);
    EVP_EncryptFinal_ex(ctx, out + out_len, &out_len);

    EVP_CIPHER_CTX_cleanup(&ctx);
```

To decrypt, the method is the same, and we've just to use *EVP_DecryptInit_ex*, *EVP_DecryptUpdate* and *EVP_DecryptFinal_ex* functions.

You'll find at [https://mkz.me/~mycroft/b/sslfilecrypt.c](https://mkz.me/~mycroft/b/sslfilecrypt.c) a simple tool using those functions in OpenSSL. It uses *getpass(3)* to get your password, then uses *sha256 functions* to get the crypt key, and then inits an IV and write in the file the crypted data.

Compilation:
```sh
$ gcc -o sslfilecrypt sslfilecrypt.c -lssl
```

Test:
```sh
$ dd if=/dev/urandom of=./test bs=512 count=8000
$ ./sslfilecrypt encrypt test test.crypted
Password: *******
$ ./sslfilecrypt decrypt test.crypted test.clear
Password: *******
$ md5sum test test.crypted test.clear
b5393e86e800f1557041f8f2d0964bef  test
5ddc5d355f49c0d2aa7140ac88430c54  test.crypted
b5393e86e800f1557041f8f2d0964bef  test.clear
$ hexdump -C test|tail -n 5
003e7fc0  a7 63 11 76 a9 85 27 ca  d1 af b8 b4 e4 21 00 b3
003e7fd0  27 ac 89 b2 c3 a1 99 1d  6a 5d 83 4c ca 27 ed 5d
003e7fe0  29 4a 50 49 f7 59 3e f2  56 a6 c5 a0 70 8e cd 89
003e7ff0  9f 0f 01 f3 93 c7 91 d9  c3 cf 1b f1 b5 3b 51 a4
$ hexdump -C test.crypted|tail -n 5
003e7ff0  f8 d8 5f a4 3d cf 3e cc  f2 97 13 2a ac 06 b6 d8
003e8000  7e 59 af 68 93 c8 ea b1  d3 6e e6 82 97 57 43 e5
003e8010  18 a4 d1 4f d5 05 58 4e  7d 34 cf 0c 8b 00 ba d4
003e8020  72 60 ea 79 cb bc 83 bb  3e 6c b2 dd 30 48 87 07
```

Note that we'll be able to change algorithm used around line 186 to use Camellia instead of AES (enabling **EVP_aes_256_cbc(3)* or *EVP_camellia_256_cbc(3)*).

This tool is a proof of concept and should not be used for a real use. It misses important things, like rewriting memory to 0 after use.

Get the [source code](https://mkz.me/~mycroft/b/sslfilecrypt.c).
