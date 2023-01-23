---
layout: post
title:  "Generate a Bitcoin address on the command line"
---
Edited: 01/23/2023

## Introduction

A bitcoin address is all you need in order for someone to send you bitcoins, and it's quite easy to generate one with only a few standard CLI commands.

To start, we can generate the secret key, in [pem](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) format, with `openssl`

```bash
openssl ecparam -name secp256k1 -genkey -noout
```

Output:

```bash
-----BEGIN EC PRIVATE KEY-----
MHQCAQEEIKoisZQEuFRDka96F+ZS8BK2vVAKEfBhNwADOlWORWcKoAcGBSuBBAAK
oUQDQgAEh6n+yJEkWqype8n+QdJUGRYP32pwgkbXoV+XpPzk1AXlaPN1L09BpdDj
GbZjVCXADjE3T1jM8g1FSqrp9zcA8Q==
-----END EC PRIVATE KEY-----
```

This randomly generated key should be __saved securely__ and kept __secret__, as it gives access to spend any bitcoins associated with it's public key.

Save this key to a file, e.g. `secret.pem`.

From this, we can retrieve the public key, again leveraging `openssl`

```bash
openssl ec -pubout -in secret.pem
```

Output:

```bash
-----BEGIN PUBLIC KEY-----
MFYwEAYHKoZIzj0CAQYFK4EEAAoDQgAEh6n+yJEkWqype8n+QdJUGRYP32pwgkbX
oV+XpPzk1AXlaPN1L09BpdDjGbZjVCXADjE3T1jM8g1FSqrp9zcA8Q==
-----END PUBLIC KEY-----
```

(Hint: Use `-conv_form compressed` to output a compressed public key)

The above command reads in the private key we saved to `secret.pem`, and outputs the public key it corresponds to, once again in pem format.

Save this public key to a file, e.g. `public.pem`

### Technical note

A Bitcoin private key is nothing but a random number defining a point on Bitcoin's [SECP256k1](https://www.secg.org/sec2-v2.pdf) elliptic curve. The public key shown above is simply a pem encoding of that point's (x, y) coordinates.

Elliptic curve cryptography leverages the fact that, while generating the public key from the private key is trivial, as seen above, it is unfeasible, with currently known computational capabilities, to calculate the private key from knowledge of the public key.

Bitcoin leverages elliptic curve cryptography, allowing for sending bitcoin to a public key, which can be freely and publicly shared, while only allowing spend access to those with knowledge of the private key.

Moving on, we can retrieve the hexadecimal representation of this key with the following

```bash
openssl ec -pubin -in public.pem -text -noout
```

Output:

```bash
pub: 
    04:87:a9:fe:c8:91:24:5a:ac:a9:7b:c9:fe:41:d2:
    54:19:16:0f:df:6a:70:82:46:d7:a1:5f:97:a4:fc:
    e4:d4:05:e5:68:f3:75:2f:4f:41:a5:d0:e3:19:b6:
    63:54:25:c0:0e:31:37:4f:58:cc:f2:0d:45:4a:aa:
    e9:f7:37:00:f1
ASN1 OID: secp256k1
```

(Hint: `openssl asn1parse` can be used to retrieve similar info)

The `04` starting byte denotes this is an uncompressed public key. The remaining bytes represent a 32 byte big endian number representing the x coordinate, followed by a 32 byte big endian number representing the y coordinate.

(Hint: Compressed public keys will start with either a `02` or `03` byte, signifying wether the y-coordinate is positive or negative, respectively, and only contain the x-coordinate.)

Let's trim this response to a concise string, by piping it into some shell commands (`grep` and `tr`), and using `echo` to print the result.

```bash
echo $(openssl ec -pubin -in public.pem -text -noout | grep -E "[a-f0-9][a-f0-9]:" | tr -d ' ' | tr -d ':' | tr -d '\n')
```

Output:

```bash
0487a9fec891245aaca97bc9fe41d25419160fdf6a708246d7a15f97a4fce4d405e568f3752f4f41a5d0e319b6635425c00e31374f58ccf20d454aaae9f73700f1
```

All the above command is doing is removing the whitespace and `:` characters for the pubkey and concatenating the result.

It is possible to use this pubkey, directly, to receive bitcoins, by using a standard pay-to-pubkey (P2PK) transaction. In fact, this was used by Satoshi in the first ever Bitcoin transaction. However, it is now uncommon to do so.

An improvement to P2PK that Satoshi offered was pay-to-pubkey-hash (P2PKH) transactions and base58check encoding.

To calculate a base58-encoded Bitcoin P2PKH address, we first hash the pubkey with sha256, followed by ripemd160.

```bash
echo 0487a9fec891245aaca97bc9fe41d25419160fdf6a708246d7a15f97a4fce4d405e568f3752f4f41a5d0e319b6635425c00e31374f58ccf20d454aaae9f73700f1 | xxd -r -p | openssl sha256 | xxd -r -p | openssl ripemd160 
```

Output:

```bash
750cdc483aef5b05f8465814b4dcc4ab36060880
```

Hashing the pubkey as such reduces the data to a 20 byte string. This helps to reduce transaction size and also offers a some extra security by not revealing the public key directly on the blockchain, until time of spenditure.

Note: The `xxd` commands above are simply used to translate the plaintext hex string to binary, to be used as input to the subsequent command.

We then prepend a zero byte, signifying that this should be used for a P2PKH transaction on Bitcoin mainnet, i.e.

```bash
00750cdc483aef5b05f8465814b4dcc4ab36060880
```  

Finally, we encode this data with [base58check](https://en.bitcoin.it/wiki/Base58Check_encoding). The "check" in "base58check" stands for the checksum that is appended for error detection purposes. We calculate that as follows. First, we take the double sha256 hash of the data.

```bash
echo 00750cdc483aef5b05f8465814b4dcc4ab36060880 | xxd -r -p | openssl sha256 | xxd -r -p | openssl sha256
```

Output:

```bash
d74bee98fffee02c7f7036a14789fd4bcc69d84f7644f732890d5ef998119b52
```

Then, we take only the first 4 bytes, i.e.

```bash
d74bee98
```

And append it to the original data, i.e.

```bash
00750cdc483aef5b05f8465814b4dcc4ab36060880d74bee98
```

This result, finally, is encoded with base58.

### Historical background

Satoshi included base58, a subset of [base64](https://en.wikipedia.org/wiki/Base64), in the [original Bitcoin source code](https://github.com/bitcoin/bitcoin/blob/v0.1.5/base58.h#L7-L12), for which the rationale of using such can be seen as follows.

```c
// Why base-58 instead of standard base-64 encoding?
// - Don't want 0OIl characters that look the same in some fonts and
//      could be used to create visually identical looking account numbers.
// - A string with non-alphanumeric characters is not as easily accepted as an account number.
// - E-mail usually won't line-break if there's no punctuation to break at.
// - Doubleclicking selects the whole number as one word if it's all alphanumeric.
```

### Base58 algorithm

The algorithm for base58 is relatively straightforward. We first remove any leading zero bytes. Then, the remaining data is interpreted as a big endian number and integer division by 58 is calculated repeatedly until there is no remainder. The value of the remainder at each iteration is mapped to a character in the bitcoin alphabet below, and the result is accumulated by prepending the character to the final base58-encoded bytestring at each iteration. Finally, any and each leading zero byte removed previously are translated to the character `1` and prepended to the final result.

The following python code performs this process.

```python
BITCOIN_ALPHABET = b"123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"

def base58encode(data):
    data = bytes.fromhex(data)
    origlen = len(data)
    data = data.lstrip(b"\x00")
    newlen = len(data)
    zeros = origlen - newlen

    encoded = b""
    integer = int.from_bytes(data, "big")
    while integer:
        integer, idx = divmod(integer, 58)
        encoded = BITCOIN_ALPHABET[idx : idx + 1] + encoded
    return (BITCOIN_ALPHABET[0:1] * zeros + encoded).decode("ascii")
```

Alternatively, we can use [base58](https://github.com/keis/base58) which can be installed using `pip`.

```bash
pip3 install base58
```

Once installed we can leverage it to retrieve our end result, a Bitcoin P2PKH address for mainnet.

```bash
echo $(echo 00750cdc483aef5b05f8465814b4dcc4ab36060880d74bee98 | xxd -r -p | base58)
```

Output:

```bash
1BfuU1LS3BDiRHtv3Tb8weAcFpFiT26YDm
```

Now this address can be used to send bitcoins to!

I have just sent 10,000 satoshis (0.00010000 BTC) as can be seen in the block explorer [here](https://blockstream.info/address/1BfuU1LS3BDiRHtv3Tb8weAcFpFiT26YDm).

It should now be trivial for anyone to take these funds (or any future funds sent to this address) with the private key data shown above, for which I will leave as an exercise for the reader 🤑. I wonder how long it will take for these funds to move... 🤔 😅

Until next time! ✌️ 😎 ❤️
