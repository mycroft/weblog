---
title:  "Accessing Bitcoin core RPCs"
date:   2018-03-04T08:56:27+0100
categories:
  - bitcoin
summary: "How to access the bitcoin core daemon using RPCs & Python"
---

Bitcoin Core client is composed of a daemon, `bitcoind`, a GUI oriented client, `bitcoin-qt` and a CLI oriented client, `bitcoin-cli`. `bitcoind` offerts an JSON-RPC interface over a HTTP socket to perform functions on the wallet, transactions, mempool, etc.

In this post, I'll quickly show how to access this interface using python and no extra library.

Firstly, you need to set RPC cookie authentification (it is also possible to set a login/password authentification, but this method will soon become obsolete as I'm writing this.). By default, no cookie is defined so one is created randomly at each `bitcoind` start, and is usable by `bitcoin-cli`. To be able to use ours, there is a script in Bitcoin Core's github repository to allow creating login credentials for a JSON-RPC user. This script is named [rpcauth.py][rpcauth]. Let's try it:

```sh
$ curl -sO https://raw.githubusercontent.com/bitcoin/bitcoin/master/share/rpcauth/rpcauth.py
$ python rpcauth.py rpc
String to be appended to bitcoin.conf:
rpcauth=rpc:3bd32b56fbdc43c8a4685f38a79498f$4432004590c837aafb86bf58962a67bda9f4aea47a3a9c0bd891465e28c44515
Your password:
1yC70vJ8-NPQ9LwFgNMf0VRxiKWavTwVLIPUetZ2kBQ=
```

You'll have to add in bitcoin.conf said line, and restart (start) bitcoind:

```sh
$ echo 'rpcauth=rpc:3bd32b56fbdc43c8a4685f38a79498f$4432004590c837aafb86bf58962a67bda9f4aea47a3a9c0bd891465e28c44515' >> $HOME/.bitcoin/testnet3/bitcoin.conf
$ /opt/bitcoin/bin/bitcoind -server -testnet -daemon -conf=/home/mycroft/.bitcoin/testnet3/bitcoin.conf
Bitcoin server starting
```

Please note that I'm using testnet on my examples.

You can test authentification using `bitcoin-cli`, using `-rpcuser` and `-rpcpassword` flags:

```sh
$ /opt/bitcoin/bin/bitcoin-cli -testnet -rpcuser=rpc -rpcpassword=aaa listaccounts
error: incorrect rpcuser or rpcpassword (authorization failed)
$ /opt/bitcoin/bin/bitcoin-cli -testnet -rpcuser=rpc -rpcpassword='1yC70vJ8-NPQ9LwFgNMf0VRxiKWavTwVLIPUetZ2kBQ=' listaccounts
{
  "": 64.33773267,
  "testaroo0": 0.00000000
}
```

We are done for this part! Still have to write some python code to connect & send commands. There is a lot of [other ways][api-reference] to reach the API, but in this example I'll just use raw `python` with only `requests` module.

```python
#!/usr/bin/env python

import requests
from requests.auth import HTTPBasicAuth
import json
import sys

class RpcCli:
    """
        Port is 8332 for mainnet, 18332 for testnet
    """
    def __init__(self, username, password, hostname='localhost', port=8332):
        self.username = username
        self.password = password
        self.hostname = hostname
        self.port = port

    def run(self, method='help', params=[]):
        url = "http://%s:%d" % (self.hostname, self.port)
        auth = HTTPBasicAuth(self.username, self.password)
        headers = {'content-type': 'application/json'}

        payload = {
            "method": method,
            "params": params,
            "jsonrpc": "1.0",
            "id": "curltext",
        }

        response = requests.post(
            url, data=json.dumps(payload), headers=headers, auth=auth)

        if response.status_code != 200:
           print(response.reason)
           return None

        result = response.json()['result']
        return self.prettyfy(result)

    def prettyfy(self, content):
        return json.dumps(content, indent=4, sort_keys=True)

    def value(self, content):
        return json.loads(content)

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: %s <command> [param1] [param2] ..." % (sys.argv[0]))
        print("Help: %s help" % (sys.argv[0]))
        sys.exit(0)

    cli = RpcCli("rpc", "1yC70vJ8-NPQ9LwFgNMf0VRxiKWavTwVLIPUetZ2kBQ=", port=18332)
    resp = cli.run(sys.argv[1], sys.argv[2:])
    print(cli.value(resp))
```

This code is pretty much straightforward: We are just doing a http query with authentification headers, and returning the result. Nothing more is required.

Let's test it:

```python
$ ./rpc-cli.py help
== Blockchain ==
getbestblockhash
getblock "blockhash" ( verbosity )
getblockchaininfo
...
$ ./rpc-cli.py listaccounts
{u'': 64.33773267, u'testaroo0': 0.0}
```

There is a lot of [api calls][api-calls] available. They go from wallet management, to stats, transaction creation, etc. More things can be done through the API than `bitcoin-qt`. Recently, I've sent some custom raw data messages on the testnet blockchain, creating my own custom transaction, in just a few lines of code using that API. I'll come back later on another article.

This API is also used for mining. Miners like [bfgminer] can connect to that API as well (if you don't use a pool/stratum server). Finally, there is some interesting examples in the [developers examples][developer-examples] wiki page.


[rpcauth]:          https://github.com/bitcoin/bitcoin/blob/master/share/rpcauth/rpcauth.py
[api-reference]:    https://en.bitcoin.it/wiki/API_reference_(JSON-RPC)
[api-calls]:        https://en.bitcoin.it/wiki/Original_Bitcoin_client/API_calls_list
[bfgminer]:         http://bfgminer.org/
[developer-examples]: https://bitcoin.org/en/developer-examples