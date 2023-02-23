---
layout: post
title:  "How to sign a Bitcoin transaction"
categories: bitcoin bitcoind bitcoin-cli openssl python python-ecdsa
---

_Edited: 02/23/2023_

## Introduction

In the last blog post, we saw how to [generate a bitcoin address on the command line](https://jtraub91.github.io/generate-a-bitcoin-address-on-the-command-line.html). This was a base58-encoded address which was used to receive bitcoin in a pay-to-pubkey-hash transaction.

In this post, you'll see how to create a transaction to send bitcoins to another address and sign it. The examples below will take place on bitcoin testnet.

### Pre-requisites

To follow along you may need the following tools

#### [Bitcoin Core](https://github.com/bitcoin/bitcoin)

We'll make use of `bitcoin-cli` configured for rpc interaction with a local `bitcoind` node

#### [OpenSSL](https://github.com/openssl/openssl)

`openssl` comes pre-installed on most systems

#### [python-ecdsa](https://github.com/tlsfuzzer/python-ecdsa)

As an alternative to signing transactions with `bitcoin-cli` and `openssl`, we'll make use of this python library.

### Create a raw transaction

I have a little bit of tBTC (testnet Bitcoin) at the address `mwDfF3Ukg81aV6ngxBQvTZWTK2ftj1Fr4T`, as can be seen by the following command.

```bash
bitcoin-cli scantxoutset start '["addr(mwDfF3Ukg81aV6ngxBQvTZWTK2ftj1Fr4T)"]'
```

Output:

```bash
{
  "success": true,
  "txouts": 28387067,
  "height": 2417653,
  "bestblock": "000000000000001c38720386eb2c9e1e5e4747cea424abe8264da9fd833614be",
  "unspents": [
    {
      "txid": "abf18fe8e1554f2d3b9826ec12de4bfc7d7ce5e04b9e4c1056c35dea54bc9727",
      "vout": 0,
      "scriptPubKey": "76a914ac3cb4fc6ee6663e697c743aad0d89fd8eb4c15f88ac",
      "desc": "addr(mwDfF3Ukg81aV6ngxBQvTZWTK2ftj1Fr4T)#8ax3vmec",
      "amount": 0.00899000,
      "height": 2412385
    }
  ],
  "total_amount": 0.00899000
}
```

How do we send these bitcoins to another address?

Well, I can create a raw transaction using `bitcoin-cli`

```bash
bitcoin-cli createrawtransaction "[{\"txid\": \"abf18fe8e1554f2d3b9826ec12de4bfc7d7ce5e04b9e4c1056c35dea54bc9727\", \"vout\": 0}]" "[{\"n3sSFbhzzefkutYmMCmzi26o5ECtbb8mCt\": 0.00898}]"
```

Output:

```bash
02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab0000000000fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000
```

In the `createrawtransaction` command above, notice I've used the `txid` and `vout` obtained from the `scantxoutset` command I ran prior. I've also specified the address (`n3sSFbhzzefkutYmMCmzi26o5ECtbb8mCt`) for the output and included a 1,000 satoshi miner fee.

(Note: In Bitcoin, unspent transaction outputs (UTXO) are always spent in full, and any amount not specified in a transaction output is assumed to be the miner fee.)

Now we need to sign the transaction.

With `bitcoin-cli` we can leverage the `signrawtransactionwithkey` method. To use this, we'll need to provide our private key in base58 [wallet import format](https://en.bitcoin.it/wiki/Wallet_import_format) (WIF). To do that, first retrieve the key in hex format. Assuming our secret key is located in `secret.pem`,

```bash
openssl asn1parse -in secret.pem
```

Output:

```bash
    0:d=0  hl=2 l= 116 cons: SEQUENCE          
    2:d=1  hl=2 l=   1 prim: INTEGER           :01
    5:d=1  hl=2 l=  32 prim: OCTET STRING      [HEX DUMP]:56C2FE62B27107CC9ADFE6CE1D919E04C356B3A7D3B518F70C28C08480645AAB
   39:d=1  hl=2 l=   7 cons: cont [ 0 ]        
   41:d=2  hl=2 l=   5 prim: OBJECT            :secp256k1
   48:d=1  hl=2 l=  68 cons: cont [ 1 ]        
   50:d=2  hl=2 l=  66 prim: BIT STRING        
```

Grab the hex string shown on the third line. This is the private key, i.e.

```bash
56C2FE62B27107CC9ADFE6CE1D919E04C356B3A7D3B518F70C28C08480645AAB
```

To convert this to base58-encoded WIF, first prepend hex `80` for mainnet or `ef` for testnet, e.g.

```bash
EF56C2FE62B27107CC9ADFE6CE1D919E04C356B3A7D3B518F70C28C08480645AAB
```

And append `01` if the private key is for a compressed public key (which this key is)

```bash
EF56C2FE62B27107CC9ADFE6CE1D919E04C356B3A7D3B518F70C28C08480645AAB01
```

Finally, encode as base58check.

```bash
echo $(echo EF56C2FE62B27107CC9ADFE6CE1D919E04C356B3A7D3B518F70C28C08480645AAB01 | xxd -r -p | base58 -c)
```

Output:

```bash
cQVMYbVNK1i6JfJm6QyvK9JBo3JxxoQq1Q4hejckbttq8V6X7wvN
```

Now with `bitcoin-cli`

```bash
bitcoin-cli signrawtransactionwithkey 02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab0000000000fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000 '["cQVMYbVNK1i6JfJm6QyvK9JBo3JxxoQq1Q4hejckbttq8V6X7wvN"]'
```

Output:

```bash
{
  "hex": "02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000006a4730440220243f86753f140847b7aef58f1724cd3b9b2699a5ea860018a9066da2cfb97b53022042469edad507502a24c81bc57e637e180fb3bf81094e82beac4ee21f795eec1c012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000",
  "complete": true
}
```

And we can verify this transaction, without actually sending it with

```bash
bitcoin-cli testmempoolaccept '["02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000006a4730440220243f86753f140847b7aef58f1724cd3b9b2699a5ea860018a9066da2cfb97b53022042469edad507502a24c81bc57e637e180fb3bf81094e82beac4ee21f795eec1c012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000"]'
```

Output:

```bash
[
  {
    "txid": "723b4eba8757701244247b14aea4a12b02c278e5447b12f85495445f67c5c749",
    "wtxid": "723b4eba8757701244247b14aea4a12b02c278e5447b12f85495445f67c5c749",
    "allowed": true,
    "vsize": 191,
    "fees": {
      "base": 0.00001000
    }
  }
]
```

Cool. 😎

Now let's see how we can sign transactions with other tools but still verify them with `bitcoin-cli`.

### Signing with `openssl`

So we have the raw transaction, i.e.

```bash
02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab0000000000fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000
```

First, let's take a look at the transaction data structure.

The following shows the transaction data with description for each part. Most values are seen as hex-encoded little endian bytes.

```bash
02000000            # tx version 2
01                  # number of tx inputs
  # txin1 txid
  2797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab
  00000000            # txin1 output number
  00                  # txin1 scriptsig length
  fdffffff            # txin1 sequence number
01                  # number of tx outputs
  d0b30d0000000000    # value
  19                  # txout1 scriptpubkey length
  # txout1 scriptpubkey
  76a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac
00000000            # tx locktime
```

### Internal Byte Order

You may notice that the txid seen in the above transaction (`2797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab`) doesn't look quite the same as that seen in the `scantxoutset` command at the beginning of this blog (`abf18fe8e1554f2d3b9826ec12de4bfc7d7ce5e04b9e4c1056c35dea54bc9727`). This is because the latter returns the txid in [rpc byte order](https://developer.bitcoin.org/glossary.html) whereas the former is referred to as [internal byte order](https://developer.bitcoin.org/glossary.html). The difference essentially amounts to a reversal of the byte order. We can convert from rpc byte order to internal byte order with the following python code.

```python
>>> txid = bytearray(bytes.fromhex("abf18fe8e1554f2d3b9826ec12de4bfc7d7ce5e04b9e4c1056c35dea54bc9727"))
>>> txid.reverse()
>>> txid.hex()
'2797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab'
```

### Preparing the signature data

One aspect that `bitcoin-cli` conceals when using `signrawtransactionwithkey` is that each input's outpoint's scriptPubKey is actually used as its scriptSig while signing.

So we're not actually signing the above transaction exactly; we'll need to insert the previous outpoint's scriptPubKey first.

If you recall, that information was provided in the output from an earlier command, e.g.

```bash
bitcoin-cli scantxoutset start '["addr(mwDfF3Ukg81aV6ngxBQvTZWTK2ftj1Fr4T)"]'
```

Output:

```bash
{
  "success": true,
  "txouts": 28387067,
  "height": 2417653,
  "bestblock": "000000000000001c38720386eb2c9e1e5e4747cea424abe8264da9fd833614be",
  "unspents": [
    {
      "txid": "abf18fe8e1554f2d3b9826ec12de4bfc7d7ce5e04b9e4c1056c35dea54bc9727",
      "vout": 0,
      "scriptPubKey": "76a914ac3cb4fc6ee6663e697c743aad0d89fd8eb4c15f88ac",
      "desc": "addr(mwDfF3Ukg81aV6ngxBQvTZWTK2ftj1Fr4T)#8ax3vmec",
      "amount": 0.00899000,
      "height": 2412385
    }
  ],
  "total_amount": 0.00899000
}
```

Including this in our raw transaction, and remembering to add the script length as hex `19` (encoded as a [compact size](https://developer.bitcoin.org/glossary.html) uint), the serialization becomes

```bash
02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000001976a914ac3cb4fc6ee6663e697c743aad0d89fd8eb4c15f88acfdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000
```

Before signing, we append `SIGHASH_ALL` (`0x01`) encoded as 4-bytes, little-endian.

```bash
02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000001976a914ac3cb4fc6ee6663e697c743aad0d89fd8eb4c15f88acfdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac0000000001000000
```

Then we take the double sha256 hash of this and sign it with `openssl` as follows.

```bash
echo 02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000001976a914ac3cb4fc6ee6663e697c743aad0d89fd8eb4c15f88acfdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac0000000001000000 | xxd -r -p | openssl sha256 | xxd -r -p | openssl sha256 -sign secret.pem | xxd -p -c 1000
```

Output:

```bash
3045022100df741c554ee34ab636a213956abf85fd4d05143026eb7b5f07ba56ac8978687902207c7e70f5790c6146599dd7c035111b93f131d95591b6030c80b1010d2e047ed3
```

Note: The signing algorithm is not deterministic and will result in different values each time for the same signature data and key

Hint: You can verify signatures with `openssl` using the `-verify` flag. E.g.

```bash
openssl sha256 -verify [file] -signature [file] [file ...]
```

Finally, we replace / insert in the original transaction, the correct scriptSig, which for pay-to-pubkeyhash transactions is denoted in Bitcoin [Script](https://en.bitcoin.it/wiki/Script) as

```bash
<sig> <pubkey>
```

Implied in the above Script are data pushes for the `<sig>` and `<pubkey>`, respectively. Also, keep in mind that `<sig>` includes a single byte for the [SIGHASH flag](https://river.com/learn/terms/s/sighash-flag/) appended to it.

With the `mwDfF3Ukg81aV6ngxBQvTZWTK2ftj1Fr4T` address corresponding to a compressed pubkey of `02726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9`, the script sig becomes.

```bash
483045022100df741c554ee34ab636a213956abf85fd4d05143026eb7b5f07ba56ac8978687902207c7e70f5790c6146599dd7c035111b93f131d95591b6030c80b1010d2e047ed3012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9
```

This entire scriptSig accounts for a length of `0x6b`. With this, the transaction becomes

```bash
02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000006b483045022100df741c554ee34ab636a213956abf85fd4d05143026eb7b5f07ba56ac8978687902207c7e70f5790c6146599dd7c035111b93f131d95591b6030c80b1010d2e047ed3012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000
```

Let's test it with `testmempoolaccept'

```bash
bitcoin-cli testmempoolaccept '["02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000006b483045022100df741c554ee34ab636a213956abf85fd4d05143026eb7b5f07ba56ac8978687902207c7e70f5790c6146599dd7c035111b93f131d95591b6030c80b1010d2e047ed3012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000"]'
```

Output:

```bash
[{'txid': '766a1be97eed0bc3f3c1e350f1c4cc2c0a634f676bedcc7cc116379f619abec6', 'wtxid': '766a1be97eed0bc3f3c1e350f1c4cc2c0a634f676bedcc7cc116379f619abec6', 'allowed': True, 'vsize': 192, 'fees': {'base': 1e-05}}]
```

Sweet. 🍭

Note: Often the signature obtained from `openssl` will not be a "canonical" signature and thus marked as invalid for the Bitcoin transaction. See [BIP62](https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki#low-s-values-in-signatures) for details. If this happens, `testmempoolaccept` may give you the following error: `mandatory-script-verify-flag-failed (Non-canonical signature: S value is unnecessarily high)`. You might have to try re-creating the signature until you get a proper s-value.

#### Signing with `python-ecdsa`

As an alternative (and potentially [less secure](https://github.com/tlsfuzzer/python-ecdsa#security)) way to sign the transaction we can use `python-ecdsa`. You can install it from [pypi](https://pypi.org/) with the following.

```bash
pip install ecdsa
```

We'll start from the raw transaction we created earlier with `bitcoin-cli`

```python
raw_tx = bytes.fromhex("02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab0000000000fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000")
```

And, of course we insert the previous outpoint's scriptPubKey for signing, as well as append the `SIGHASH_ALL` bytes.

```python
sig_data = bytes.fromhex("02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000001976a914ac3cb4fc6ee6663e697c743aad0d89fd8eb4c15f88acfdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac0000000001000000")
```

From here, like before, we take the double sha256 hash and sign it. We can use python's [hashlib](https://docs.python.org/3/library/hashlib.html) for taking sha256, and we'll use `python-ecdsa` to sign.

```python
>>> import hashlib
>>> import ecdsa
>>> sig_data = hashlib.sha256(hashlib.sha256(sig_data).digest()).digest()
>>> with open("secret.pem", "rb") as pem_file:
...     pem = pem_file.read()
...
>>> sk = ecdsa.SigningKey.from_pem(pem)
>>> sig = sk.sign_digest(sig_data, sigencode=ecdsa.util.sigencode_der)
>>> sig.hex()
'3044022064ca8305562c788a7f7b17d68a2c0141f7b62914e7f2950fe043f01bfbbb99e902201179d318d6332b871827b8fb310b013f19401c6ea74adbde990e466c736285a7'
```

Again, we take this signature and the previous outpoint's pubkey to form the scriptSig

```python
>>> SIGHASH_ALL = 0x01
>>> pubkey = bytes.fromhex("02726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9")
>>> p2pkh_script_sig = (len(sig) + 1).to_bytes(1, "little") + sig + SIGHASH_ALL.to_bytes(1, "little") + len(pubkey).to_bytes(1, "little") + pubkey
>>> p2pkh_script_sig.hex()
'473044022064ca8305562c788a7f7b17d68a2c0141f7b62914e7f2950fe043f01bfbbb99e902201179d318d6332b871827b8fb310b013f19401c6ea74adbde990e466c736285a7012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9'
```

Now finally, we replace this scriptSig in the original transaction (including the OP_PUSHDATA of hex `6a` for its length).

```bash
02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000006a473044022064ca8305562c788a7f7b17d68a2c0141f7b62914e7f2950fe043f01bfbbb99e902201179d318d6332b871827b8fb310b013f19401c6ea74adbde990e466c736285a7012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000
```

And test it with `bitcoin-cli`'s `testmempoolaccept`

```bash
bitcoin-cli testmempoolaccept '["02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000006a473044022064ca8305562c788a7f7b17d68a2c0141f7b62914e7f2950fe043f01bfbbb99e902201179d318d6332b871827b8fb310b013f19401c6ea74adbde990e466c736285a7012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000"]'
```

Output:

```bash
[{'txid': 'e110d9ab2c5af39b35724f1a9de28379f3c4dd0c0ceb96c44e87787351b567b4', 'wtxid': 'e110d9ab2c5af39b35724f1a9de28379f3c4dd0c0ceb96c44e87787351b567b4', 'allowed': True, 'vsize': 191, 'fees': {'base': 1e-05}}]
```

Nice. 🤓

### Conclusion

And there we have it. Just by using some open source tooling we can create keys, generate Bitcoin addresses, create raw transactions, and sign them for spenditure. We got somewhat lucky with `openssl` and `python-ecdsa` by generating canonical signatures on the first try. If we hadn't, we could have simply re-tried until achieveing one.

#### Final thoughts on `python-ecdsa` security

You may ask why I chose to show the use of `python-ecdsa` if it is potentially [less secure](https://github.com/tlsfuzzer/python-ecdsa#security). Honestly, I'm not sure 😅. I sort of wanted to document how to do it with this Python library anyways 🤷‍♂️.  Also, candidly, I don't completely understand the security risks in `python-ecdsa`'s README. I think I understand what a [side-channel attack](https://en.wikipedia.org/wiki/Side-channel_attack) _is_, but how much of a concern is it really? Does `openssl` provide security against these attacks? What about other common Bitcoin wallets and libraries? Furthermore, `python-ecdsa`'s README proclaims that any pure Python implementation would be vulnerable since "Python does not provide side channel secure primitives". That seems like a bold accusation. Is it true? At the end of the day, I don't know, and I suppose I will have to the leave it to the cryptographic experts for now. If side-channel secure programming is indeed "impossible" in pure Python, I would hope there'd be some effort to fix or improve that. Either way, I'm hoping to stir some conversation from the community with this blog.

#### Actually sending the transaction on Bitcoin Testnet

Finally, I'll go ahead and actually send the transaction on Testnet (using the transaction signed via `openssl` shown above)

```bash
bitcoin-cli sendrawtransaction "02000000012797bc54ea5dc356104c9e4be0e57c7dfc4bde12ec26983b2d4f55e1e88ff1ab000000006b483045022100df741c554ee34ab636a213956abf85fd4d05143026eb7b5f07ba56ac8978687902207c7e70f5790c6146599dd7c035111b93f131d95591b6030c80b1010d2e047ed3012102726b45a5b1b506015dc926630b2627454d635d87eeb72bb7d5476d545d6769f9fdffffff01d0b30d00000000001976a914f5326ca3988fd949f3b7699922caf3eadc86e71488ac00000000"
```

Output:

```bash
766a1be97eed0bc3f3c1e350f1c4cc2c0a634f676bedcc7cc116379f619abec6
```

I hope y'all enjoyed this blog. Stay tuned for more content! ✌️
