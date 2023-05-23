---
layout: post
title:  Introducing bits
categories: bitcoin python cli
author: Jason Traub
---

Introducing [bits](https://github.com/jtraub91/bits), a free and open source cli tool and pure Python library developed for Bitcoin.

This post will serve as informal documentation for some of what you can do with [bits](https://github.com/jtraub91/bits) today, and what is planned for the future.

## Installation

```bash
git clone https://github.com/jtraub91/bits
cd bits/
pip install .
```

## Usage

[bits](https://github.com/jtraub91/bits) is pure Python, and thus, can be used as a library by calling its various functions and methods directly (with documentation coming soon 🚀); however, often it's convenient to simply use a command line interface (CLI) to common functionality. The [bits](https://github.com/jtraub91/bits) CLI provides many different methods which are explained in detail in the subsequent sections.

## Command Line Interface

### The `bits` base command

The `bits` command, by itself, with no subcommand present, is a command in its own rite which will "convert input bits to bytes".

To elaborate, three input / output (I/O) formats are supported: raw bytes, binary string, and hexadecimal string, which may be specified independently making `bits` able to be used as a converter between these formats. Raw bytes are read transparently; however, binary string and hexadecimal string input is stripped of leading or trailing whitespace, _and_ left zero padded to the nearest byte if not provided as an integer multiple of 8 bits. Furthermore, raw bytes are output as-is, yet binary and hexadecimal string will be output will have a trailing newline character.

#### Comparing to `xxd`

The `bits` base command, sharing functionality with standard Linux utility [xxd](https://linux.die.net/man/1/xxd), can be used similarly in some respects.

With `xxd`, we can convert a raw binary input to a hex string like so,

```bash
head -c 8 /dev/urandom | xxd -p
```

Similarly we can do the following with `bits`

```bash
head -c 8 /dev/urandom | bits -1 -0x
```

The `-1` indicates that the _input_ is raw bytes, while the `-0x` flag indicates the _output_ shall be a hexadecimal string. In general, these flags can be ommitted, but unless otherwise [configured](https://github.com/jtraub91/bits#configuration), the default I/O format for most commands is hexadecimal string.

The `bits` command will output the string on a single line, whereas with `xxd` we may have to use the `-c` argument to achieve a similar result. For example,

```bash
head -c 32 /dev/urandom | xxd -p -c 32
```

becomes simply

```bash
head -c 32 /dev/urandom | bits -1 
```

(As alluded to above, with `-0x` omitted, and no other configuration, the output format defaults to hexadecimal string)

Furthermore, the support for binary strings provides functionality that `xxd` does not (nor any other CLI tool that I am aware of), in that it can convert between binary string representation. For example, with `xxd` in `-bits` (`-b`) mode, we can do,

```bash
head -c 32 /dev/urandom | xxd -b
```

to get a binary string representation of the input; but it outputs it in octets, which may be inconvenient for further processing. Furthermore, the `-revert` (`-r`) flag "[does] not work with [`-bits`] mode". [[1]](#citations)

So with `xxd` we can convert raw bytes to a hexadecimal string, and revert.

```bash
head -c 32 /dev/urandom | xxd -p -c 32 | xxd -r -p
```

But, we cannot do the same with `-b`

```bash
head -c 32 /dev/urandom | xxd -b | xxd -r -b
xxd: sorry, cannot revert this type of hexdump
```

But, we can with `bits`

```bash
head -c 32 /dev/urandom | bits -1 -0b | bits -1b -0
```

### CLI subcommands

Now, let's get into some of the more useful and specific Bitcoin functions that `bits` provides. These are provided by various _subcommands_. You can view a full list of available subcommands with

```bash
bits -h
```

Subcommand specific help is also available with

```bash
bits <subcommand> -h
```

### Generate a Bitcoin private key

The `bits key` command will generate a bitcoin private key, which is nothing but a 256 bit random value strictly less than `SECP256K1_N`. [[2]](#citations)

This, you may notice, _can_ be generated with the base command, e.g.

```bash
head -c 32 /dev/urandom | bits -1
```

but instead of having to manually confirm the output is below `SECP256K1_N` you can use `bits key` instead for convenience.

Also, note that if a command has input or output, it will accept an `--input-file` (`-in`, `-i`) and/or `--out-file` (`-out`, `-o`) optional argument, defaulting to stdin or stdout, respectively. (Also, as applicable, a command with input or output will usually include the input and output format flags illustrated above for the [bits base command](#the-bits-base-command)). Thus, to conceal the private key from being exposed on the command line, you may use the following command, for example.

```bash
bits key -out secret.hex
```

(or standard shell redirection, e.g. `bits key > secret.hex`)

For compatibility with legacy tooling (if you're into that sort of thing), the `bits key` command supports PEM-encoded private key output, e.g.

```bash
bits key -0pem
```

Note that `bits` _does not_ currently provide support for PEM-encoded _input_ on the CLI, so you will need to convert back to a supported format (raw, bin, hex) in order to use with other `bits` (sub)commands. This can be done with [openssl](https://www.openssl.org/) (`openssl ec` or `openssl asn1parse`, for example).

### Create a public key from a private key

The `bits pubkey` command generates a public key (pubkey) from a private key. E.g.

```bash
bits pubkey -in secret.hex
```

You may output a compressed pubkey with the `--compressed` (`-X`) flag.

```bash
bits pubkey -in secret.hex -X -out pubkey.hex
```

Notice we leverage the `-out` flag again above to save the pubkey to a file.

### Creating a bitcoin address

Now, you may want to create a Bitcoin address from your key pair. The `bits addr` command provides a convenient method to handle the encoding for various address types. This command supports generating legacy and segwit addresses alike, but expects the payload to passed in as the argument. For example, for legacy [pay to pubkey hash](https://learnmeabitcoin.com/technical/p2pkh) addresses (P2PKH), we'll need to hash the pubkey first with `hash160`. The [bits](https://github.com/jtraub91/bits) CLI fortunately provides the `ripemd160`, `sha256`, `hash160`, and `hash256` crypto hashing subcommands for convenience. Thus, to generate a legacy Bitcoin address from our pubkey, we may use the following, for example.

```bash
bits hash160 -in pubkey.hex | bits addr | awk '{print $1}'
```

In the above example, the pubkey saved to `pubkey.hex` is read as input to `bits hash160`, thus is hashed, and piped to the `bits addr` command to complete the encoding. The output of `bits addr` is then piped into `awk` only for readability, i.e. to produce a newline on the shell output since `bits addr` will technically output raw binary, with no newline appended; however, in these situations `bits` provides the `--print` (`-P`) flag to do the same, for convenience. E.g.

```bash
bits hash160 -in pubkey.hex | bits addr -P
```

The `bits addr` command accepts a `--type` (`-T`) flag for address type which defaults to "p2pkh" for P2PKH addresses, but may be specified to be "p2sh" for [BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki) [pay to script hash](https://learnmeabitcoin.com/technical/p2sh) (P2SH) addresses.

A testnet address may be generated instead by leveraging the `--network` (`-N`) flag. E.g.

```bash
bits hash160 -in pubkey.hex | bits addr -N testnet -P
```

Segwit addresses can be generated similarly, e.g.

```bash
bits hash160 -in pubkey.hex | bits addr --witness-version 0 -P
```

The presence of the `--witness-version` (`--wv`) flag indicates a segwit address and its value represents the version. When this flag is present the `--type` flag is ignored. Segwit addresses, per [BIP 141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki), identify themselves as [pay to witness key hash](https://river.com/learn/terms/p/p2wpkh/) (P2WKH) or [pay to witness script hash](https://river.com/learn/terms/p/p2wsh/) (P2WSH) by the size of their payload, thus the user is expected to form and provide the correct payload as necessary.

The `bits addr` command isn't doing anything magical 🧙‍♂️ It accepts a payload and optional network and address type (or witness version) arguments, prepends the correct version byte and encodes in [base58check](https://en.bitcoin.it/wiki/Base58Check_encoding) or [bech32](https://en.bitcoin.it/wiki/Bech32) ([BIP 173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)), as appropriate. This can also be done manually with other [bits](https://github.com/jtraub91/bits) CLI subcommands as we will see in the next section.

### Encoding (and decoding) with `base58` and `bech32`

The [bits](https://github.com/jtraub91/bits) CLI provides methods for encoding and decoding base58(check) and bech32.

Recreating what was performed above by the `bits addr` command, but instead with `bits base58` and `bits bech32`, repectively, we could do the following, for example.

For legacy P2PKH addresses:

- Take your pubkey and hash it like before
- Pre-prend the version byte (`0x00` for mainnet)
- Encode with [base58check](https://en.bitcoin.it/wiki/Base58Check_encoding)

Doing this, all at once, on the command line, would look something like the following.

```bash
bits hash160 -in pubkey.hex | awk '{print "00"$1}' | bits base58 --check
```

The `--check` option is supplied so that the checksum is appended. The `bits base58` command also supports a `--decode` option for retrieving the original payload, e.g.

```bash
echo -n <base58-check-encoded-address> | bits base58 --check --decode
```

The `--check` flag with the `--decode` flag present will make `bits` ensure that the checksum is valid.

Similarly, we can reproduce segwit address we previously created with `bits addr`, instead with `bits bech32`.

```bash
bits hash160 -in pubkey.hex | bits bech32 --hrp bc --wv 0 -P
```

As we can see, the `bits bech32` command gives us a little bit more flexibility with regard to the [bech32 specification](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki#bech32), whereas `bits addr` provides practical convenience. With `bits bech32` we can supply an arbitrary human readable part via `--hrp` and an optional Segwit version via the `--witness-version` (`--wv`) flag. From there, the data passed as input is encoded, with checksum appended. Alternatively, the `bits addr` command provides convenience by performing the address encoding as inferred from the options provided, i.e. `--network` (`-N`), `--type` (`-T`), and `--witness-version` (`--wv`).

Furthermore, we can also _decode_ bech32, e.g.

```bash
echo -n <bech32-encoded> | bits bech32 --decode
```

The output of the previous commnd is a JSON string. If input is determined to be a segwit address, the output will have the keys "network", "witness_version", and "witness_program"; otherwise the keys will be "hrp" and "payload".

### Mnemonic seeds and HD wallet operations

`bits` provides two subcommands for mnemonic / seed generation and hierarchical deterministic wallets per [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki), [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki), and [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki): `bits mnemonic` and `bits hd`.

To generate a mnemonic seed phrase you may simply use the following.

```bash
bits mnemonic
```

This command also supports providing your own entropy. For example,

```bash
head -c 32 /dev/urandom | bits mnemonic -1 --from-entropy
```

Furthermore, we can provide the generated mnemonic as input to the `bits mnemonic` command, with the `--to-seed` or `--to-master-key` options present to generate the seed or master key (per [BIP 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#master-key-generation)), respectively. These options will cause `bits mnemonic` to prompt for a passphrase at the CLI (which can be blank).

```bash
echo <mnemonic-phrase> | bits mnemonic --to-master-key
passphrase: 
```

The base58check-encoded master key, provided as output of the previous command, is needed for further derivation with the `bits hd` command.

With `bits hd` you may derive private extended private keys (xprv) or extended public keys (xpub) by supplying the root key and a derivation path per [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki), and/or [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki). For example,

```bash
echo -n <root-key> | bits hd "m/44'/0'/0'/0/0"
```

The leading `m` indicates private key derivation, implying that a xprv must be supplied as input. The command will then derive and return the xprv at the derivation path provided, _but_ the xpub can be returned instead by supplying the `--xpub` flag. E.g.

```bash
echo -n <root-key> | bits hd "m/44'/0'/0'/0/0" --xpub
```

Also, a deserialized key can be emitted to stderr as a JSON string (with keys "version", "depth", "parent_key_fingerprint", "child_no", and "key") by supplying the optional `--dump` flag.

Public derivation is supported and can be used by indicating a capital `M` in the derivation path. E.g.

```bash
echo -n <xpub> | bits hd "M/0" 
```

Also, of note, is that you aren't required to use the _root_ master key as input for the derivation. Any derived key can be used as the starting point, and derivation will happen relative to it. This may be useful in practice, so as to not expose the root master key on the same machine that derivation is performed.

### Leveraging a Bitcoin Core Node

So far we've seen that the [bits](https://github.com/jtraub91/bits) CLI is able to do some basic bits to bytes manipulations and provide some utilites for generating and managing Bitcoin keys, addresses, and whatnot. What [bits](https://github.com/jtraub91/bits) _does not_ do, currently, is provide a full node for consensus. For now, I recommend having a local [Bitcoin Core](https://github.com/bitcoin/bitcoin) node running (i.e. `bitcoind`) so that blockchain data can be downloaded, transactions can be relayed, and a node for verifying and maintaining consensus can be employed, strengthening the Bitcoin network.

Part of the plan for [bits](https://github.com/jtraub91/bits) is to implement a full node, with built-in [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) server, by version 1. There does actually exist code in the codebase that has started to implement such, but it is not currently very functional nor supported.

So in the meantime, I recommend leveraging [Bitcoin Core](https://github.com/bitcoin/bitcoin) here. As we continue thru this blog, you may want a [Bitcoin Core](https://github.com/bitcoin/bitcoin) node while using subcommands such as `bits tx`, for looking up necessary blockchain data, but additionally and moreover, `bits` can be configured to directly interact with [Bitcoin Core](https://github.com/bitcoin/bitcoin) over RPC, either by sending raw RPC commands via the `bits rpc` command, or by leveraging it internally, as with the `bits send` command, for example. These will be explained, in detail, in the following sections.

To download a [Bitcoin Core](https://github.com/bitcoin/bitcoin) binary for installation on your OS, see [downloads](https://bitcoincore.org/en/download/).

### Configuring `bits` with a local Bitcoin Core node

Nominal bits configuration is located in your user's home directory, in a directory named `.bits`, in a file named either `config.toml` or `config.json`, depending on [config file support](https://github.com/jtraub91/bits#config-file-support).

You may configure a connection via RPC with either cookie based authentication or a user & password. To connect via cookie based authentication, all you should configure is `rpc_url` and `rpc_datadir`. These may be specified at the CLI for dependent subcommands, or configured in `~/.bits/config.toml`, for example.

For cookie based auth,

```toml
rpc_url="http://localhost:8332"
rpc_datadir="~/.bitcoin"
```

Or for rpc user / password,

```toml
rpc_url="http://localhost:8332"
rpc_user="user"
rpc_password="password"
```

Note: If `rpc_datadir` is present, cookie based auth will be used. If you desire to use rpc user / password instead, `rpc_datadir` should be absent or left blank.

Once we have a configured rpc node, we can issue commands to it directly, as we would with `bitcoin-cli`. E.g.

```bash
bits rpc getdifficulty
```

Now, let's see how we can form transactions with the [bits](https://github.com/jtraub91/bits) CLI, and relay them via the local [Bitcoin Core](https://github.com/bitcoin/bitcoin) node we've configured above.

### Creating raw transactions

Raw Bitcoin transactions can be created with the `bits tx` command. As with any subcommand, you may use `-h` (e.g. `bits tx -h`) for information on the available options, but the key principle of this command is that it expects any number of `--txin` (`-txin`) and `--txout` (`-txout`) flags each with an argument provided as a JSON string with the keys "txid", "vout", and "scriptsig" or  "satoshis" and "scriptpubkey", respectively.

For example, suppose we have the following [unspent transaction output](https://academy.binance.com/en/glossary/unspent-transaction-output-utxo) (UTXO), associated with the `mkmnWvK9fKEQdUcYrpS7gNkxBZ4CACLmKd` address.

```bash
bits rpc scantxoutset start '["addr(mkmnWvK9fKEQdUcYrpS7gNkxBZ4CACLmKd)"]' | jq .unspents
```

Output:

```bash
[
  {
    "txid": "1c9eb55b17705ca5fc3047d9490ebd3e38422be18fa130dc2389b233767dbc65",
    "vout": 0,
    "scriptPubKey": "76a91439a6b4b85e0108176524ab3b1584f5b9ac21d6da88ac",
    "desc": "addr(mkmnWvK9fKEQdUcYrpS7gNkxBZ4CACLmKd)#vmpyq35x",
    "amount": 50,
    "height": 109
  }
]
```

We can use the "txid" and "vout" shown above, to find more information on this UTXO again using our configured [Bitcoin Core](https://github.com/bitcoin/bitcoin) RPC connection.

```bash
bits rpc gettxout 1c9eb55b17705ca5fc3047d9490ebd3e38422be18fa130dc2389b233767dbc65 0 | jq .
```

Output:

```bash
{
  "bestblock": "2c9574a9db1c9b2178a539e7455e0b84aee06bd5b77e36ae65a89457d96e3e16",
  "confirmations": 101,
  "value": 50,
  "scriptPubKey": {
    "asm": "OP_DUP OP_HASH160 39a6b4b85e0108176524ab3b1584f5b9ac21d6da OP_EQUALVERIFY OP_CHECKSIG",
    "desc": "addr(mkmnWvK9fKEQdUcYrpS7gNkxBZ4CACLmKd)#vmpyq35x",
    "hex": "76a91439a6b4b85e0108176524ab3b1584f5b9ac21d6da88ac",
    "address": "mkmnWvK9fKEQdUcYrpS7gNkxBZ4CACLmKd",
    "type": "pubkeyhash"
  },
  "coinbase": true
}
```

This will show information on the UTXO, including the [scriptPubKey](https://learnmeabitcoin.com/technical/scriptPubKey) in assembly (asm) format. Understanding that the UTXO is a P2PKH output will be important as we continue forming the raw transaction, but first, let's see how we can determine this, using the `bits script` command, instead of leveraging our configured [Bitcoin Core](https://github.com/bitcoin/bitcoin) RPC connection.

### Encoding (and decoding) Script with `bits script`

We can _decode_ the scriptPubKey shown above, with the `bits script` command by using the `--decode` flag. E.g.

```bash
bits script 76a91439a6b4b85e0108176524ab3b1584f5b9ac21d6da88ac --decode
```

Output:

```bash
[["OP_DUP", "OP_HASH160", "39a6b4b85e0108176524ab3b1584f5b9ac21d6da", "OP_EQUALVERIFY", "OP_CHECKSIG"]]
```

We'll make more use of `bits script` in a bit (no pun intended 😉), but let's first recap where we are at in forming the raw transaction with `bits tx`.

From either of these prior commands, we learn that the UTXO is a P2PKH output. We need this information to understand how to build the pre-signature transaction image, as well as, form the correct final [scriptSig](https://river.com/learn/terms/s/scriptsig/) to unlock the funds. But, we will also need to specify our transaction outputs, indicating the amount of Bitcoin to send and to which address. With Bitcoin, UTXOs are always spent in full, so it is common to have one transaction output indicating our recipient's address, and another to send the "change" back to an address owned by the sender, sometimes referred to as the "change address". Also, be aware that this implies that any amount available from a UTXO which is _not_ used in any of the transaction outputs may be assumed as the "miner fee".  

The UTXO above has 50 BTC (5,000,000,000 satoshis). Let's send 4,000,000,000 satoshis to `mp6UyrvbWH3sq1WRjHnVZr23aJfuwDktZg` and use `msgrvCT4DagqPAuFoqqCWqBouv43KyRors` as the change address, leaving 1000 satoshis available for the miner fee. Remember txouts are supplied as a JSON string with keys "satoshis" (specifying the amount) and "scriptpubkey" (specifying the unlocking script). The addresses `mp6UyrvbWH3sq1WRjHnVZr23aJfuwDktZg` and `msgrvCT4DagqPAuFoqqCWqBouv43KyRors` imply a P2PKH transaction output, so to create the scriptpubkey, we can again use `bits script`. `bits script -h` includes useful information on how to form various standard transaction scripts.

The pubkey hash that corresponds to  `mp6UyrvbWH3sq1WRjHnVZr23aJfuwDktZg` and `msgrvCT4DagqPAuFoqqCWqBouv43KyRors` is `5e185942ab1fb90f4880c57384df78dc657a0dba` and `85812e40cf29f7103731cc6a17b101a4d640aec0`, respectively. (This information can be found with `bits base58 --check --decode`).

Thus, for the transaction output spending to `mp6UyrvbWH3sq1WRjHnVZr23aJfuwDktZg`, the scriptPubkey becomes

```bash
bits script OP_DUP OP_HASH160 5e185942ab1fb90f4880c57384df78dc657a0dba OP_EQUALVERIFY OP_CHECKSIG
```

Output:

```bash
76a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac
```

And for `msgrvCT4DagqPAuFoqqCWqBouv43KyRors`,

```bash
bits script OP_DUP OP_HASH160 85812e40cf29f7103731cc6a17b101a4d640aec0 OP_EQUALVERIFY OP_CHECKSIG
```

Output:

```bash
76a91485812e40cf29f7103731cc6a17b101a4d640aec088ac
```

Putting it all together, and forming the pre-signature transaction data we have,

```bash
bits tx -txin '{"txid": "1c9eb55b17705ca5fc3047d9490ebd3e38422be18fa130dc2389b233767dbc65", "vout": 0, "scriptsig": "76a91439a6b4b85e0108176524ab3b1584f5b9ac21d6da88ac"}' -txout '{"satoshis": 4000000000, "scriptpubkey": "76a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac"}' -txout '{"satoshis": 999999000, "scriptpubkey": "76a91485812e40cf29f7103731cc6a17b101a4d640aec088ac"}'
```

Output:

```bash
010000000165bc7d7633b28923dc30a18fe12b42383ebd0e49d94730fca55c70175bb59e1c000000001976a91439a6b4b85e0108176524ab3b1584f5b9ac21d6da88acffffffff0200286bee000000001976a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac18c69a3b000000001976a91485812e40cf29f7103731cc6a17b101a4d640aec088ac00000000
```

Notice that the UTXO's scriptpubkey is inserted as the scriptsig for the transaction data that used for signing. Now finally, to spend the transaction input, we must sign the transaction data and re-insert the correct scriptsig. For this, we will make use of the `bits sig` command.

### Signing, and verifying signatures

With the `bits sig` command, we can create (and verify) Bitcoin signatures.

To sign the above transaction with the private key associated with the UTXO address (i.e. `sender.hex`), we can use following command.

```bash
bits sig -i sender.hex 010000000165bc7d7633b28923dc30a18fe12b42383ebd0e49d94730fca55c70175bb59e1c000000001976a91439a6b4b85e0108176524ab3b1584f5b9ac21d6da88acffffffff0200286bee000000001976a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac18c69a3b000000001976a91485812e40cf29f7103731cc6a17b101a4d640aec088ac00000000 --sighash all
```

Output:

```bash
3045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a901
```

If you would need to verify this signature with the sender's pubkey (i.e. `sender.pub`), the following may be used.

```bash
echo 02ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fc | bits sig --verify 010000000165bc7d7633b28923dc30a18fe12b42383ebd0e49d94730fca55c70175bb59e1c000000001976a91439a6b4b85e0108176524ab3b1584f5b9ac21d6da88acffffffff0200286bee000000001976a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac18c69a3b000000001976a91485812e40cf29f7103731cc6a17b101a4d640aec088ac00000000 --signature 3045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a901
```

Output:

```bash
OK
```

Now that we have the signature, we can form the final scriptsig necessary for this P2PKH UTXO. The scriptsig for a P2PKH output is `<signature> <pubkey>`, both of which can be found above. Therefore we have,

```bash
bits script 3045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a901 02ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fc
```

Output:

```bash
483045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a9012102ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fc
```

And now, forming the final transaction

```bash
bits tx -txin '{"txid": "1c9eb55b17705ca5fc3047d9490ebd3e38422be18fa130dc2389b233767dbc65", "vout": 0, "scriptsig": "483045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a9012102ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fc"}' -txout '{"satoshis": 4000000000, "scriptpubkey": "76a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac"}' -txout '{"satoshis": 999999000, "scriptpubkey": "76a91485812e40cf29f7103731cc6a17b101a4d640aec088ac"}'
```

Output:

```bash
010000000165bc7d7633b28923dc30a18fe12b42383ebd0e49d94730fca55c70175bb59e1c000000006b483045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a9012102ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fcffffffff0200286bee000000001976a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac18c69a3b000000001976a91485812e40cf29f7103731cc6a17b101a4d640aec088ac00000000
```

Finally, this transaction is ready to be broadcasted to the Bitcoin network. And we can send it using our [configured](#configuring-bits-with-a-local-bitcoin-core-node) [Bitcoin Core](https://github.com/bitcoin/bitcoin) RPC node. First let's test the transaction's validity with,

```bash
bits rpc testmempoolaccept '["010000000165bc7d7633b28923dc30a18fe12b42383ebd0e49d94730fca55c70175bb59e1c000000006b483045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a9012102ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fcffffffff0200286bee000000001976a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac18c69a3b000000001976a91485812e40cf29f7103731cc6a17b101a4d640aec088ac00000000"]'
```

Output:

```bash
[{'txid': '4ecc3ff0cc54a09b71e4fe5b981bcc930a31e09dfef7f8804b7be4932b251dd2', 'wtxid': '4ecc3ff0cc54a09b71e4fe5b981bcc930a31e09dfef7f8804b7be4932b251dd2', 'allowed': True, 'vsize': 226, 'fees': {'base': 1e-05}}]
```

And then actually send it with

```bash
bits rpc sendrawtransaction "010000000165bc7d7633b28923dc30a18fe12b42383ebd0e49d94730fca55c70175bb59e1c000000006b483045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a9012102ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fcffffffff0200286bee000000001976a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac18c69a3b000000001976a91485812e40cf29f7103731cc6a17b101a4d640aec088ac00000000"
```

Output:

```bash
4ecc3ff0cc54a09b71e4fe5b981bcc930a31e09dfef7f8804b7be4932b251dd2
```

Remember, you will need to wait, at least until the transaction is mined in a block, for the transfer of funds to be reflected in the blockchain.

#### Decoding raw transactions

You may also _decode_ this raw transaction using `bits tx --decode`. E.g.

```bash
echo 010000000165bc7d7633b28923dc30a18fe12b42383ebd0e49d94730fca55c70175bb59e1c000000006b483045022100f0cbe27a8f8fe3bde49abcfb15c218fd0e9afe1a69354976752cc4c3c0b11ab60220303ca69643fa29dad33868b1fc8ee1fad29b41890c0cc25f7393a610e6a2d9a9012102ed5df88a8fa1f10389b263eeae8df6456fc16f38da21d8737b9a65a246c358fcffffffff0200286bee000000001976a9145e185942ab1fb90f4880c57384df78dc657a0dba88ac18c69a3b000000001976a91485812e40cf29f7103731cc6a17b101a4d640aec088ac00000000 | bits tx --decode | jq .
```

### A more convenient approach for sending funds: `bits send`

Forming raw transactions as above, affords us great flexibility, but is a bit tedious. Therefore [bits](https://github.com/jtraub91/bits) provides the `bits send` command, which, as opposed to `bits tx`, provides a better user inteface, and does some of the tedious steps behind the scenes; thereby trading flexibility for convenience.

The `bits send` command integrates directly with (and thus depends on) a [configured](#configuring-bits-with-a-local-bitcoin-core-node) [Bitcoin Core](https://github.com/bitcoin/bitcoin) node.

Let's see how it works.

Suppose we have 3.125 BTC associated with the SegWit address `bcrt1quzz3xxt2p0488hvlfelj78dq2pgcsjkd3auhwz` on a local regtest Bitcoin network. Let's send half of these funds to a new address, `bcrt1qddfhzmx80snsgudw8hr2qj9v3yz0vgd9wj5wca`, and use `mp6nDfuUJWV1pNdcxNQtbU29eg3fJ6eK4c` as the change address.

Many of the tedious steps we had to do in [creating raw transactions](#creating-raw-transactions) can now be simply expressed as the following.

```bash
bits send bcrt1quzz3xxt2p0488hvlfelj78dq2pgcsjkd3auhwz bcrt1qddfhzmx80snsgudw8hr2qj9v3yz0vgd9wj5wca --send-fraction 0.5 --change-addr mp6nDfuUJWV1pNdcxNQtbU29eg3fJ6eK4c
```

Output:

```bash
010000000189592b942d94a2265c4c986560363b5cf672fd9c6b8b5acc212759e436efa6150000000000ffffffff02a82b5009000000001600146b53716cc77c270471ae3dc6a048ac8904f621a5902f5009000000001976a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac00000000
```

We can inspect this output by using a command seen previously, `bits tx --decode`, i.e.

```bash
echo 010000000189592b942d94a2265c4c986560363b5cf672fd9c6b8b5acc212759e436efa6150000000000ffffffff02a82b5009000000001600146b53716cc77c270471ae3dc6a048ac8904f621a5902f5009000000001976a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac00000000 | bits tx --decode | jq .
```

Output:

```bash
{
  "txid": "1053c3fe474b13156ad6611d5ebfaf6c500711f7742d5978ad3559d2b8beeeb3",
  "wtxid": "1053c3fe474b13156ad6611d5ebfaf6c500711f7742d5978ad3559d2b8beeeb3",
  "version": 1,
  "txins": [
    {
      "txid": "89592b942d94a2265c4c986560363b5cf672fd9c6b8b5acc212759e436efa615",
      "vout": 0,
      "scriptsig": "",
      "sequence": "ffffffff"
    }
  ],
  "txouts": [
    {
      "value": 156249000,
      "scriptpubkey": "00146b53716cc77c270471ae3dc6a048ac8904f621a5"
    },
    {
      "value": 156250000,
      "scriptpubkey": "76a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac"
    }
  ],
  "locktime": 0
}
```

By inspecting the output above, we see 156249000 satoshis in a txout associated with a scriptpubkey indicating a payment to the `bcrt1qddfhzmx80snsgudw8hr2qj9v3yz0vgd9wj5wca` address we used above. This corresponds to half the original amount associated with the sender address of `bcrt1quzz3xxt2p0488hvlfelj78dq2pgcsjkd3auhwz`, minus the miner fee (which defaults to 1000 satoshis but may be specified by using the `--miner-fee` option). We also see the rest of the satoshis being associated in a txout with a scriptpubkey indicating a payment to to our change address of `mp6nDfuUJWV1pNdcxNQtbU29eg3fJ6eK4c`.

However, this raw transaction has not been signed yet. We can see that from the above output by noticing that there is no witness data, which is needed for spenditure from the SegWit sender address, and correspondingly the wtxid is equal to the txid. But, we can do so easily with the `bits send` command, by provided the private key(s) necessary for spenditure, in _extended_ [WIF](https://en.bitcoin.it/wiki/Wallet_import_format) format. The extended WIF encoded private key can be created with the `bits wif` command.

### Encode (or decode) a private key in extended WIF format

Building upon WIF and code seen in the [electrum wallet codebase](https://github.com/spesmilo/electrum/blob/4.4.0/electrum/bitcoin.py#L618-L625), _extended_ WIF, allows for encoding a private key with additional data indicating not only its corresponding address type, but also the additional data necessary for spenditure (i.e. the data needed for creating an appropriate scriptsig, such as the redeem script for a P2SH address, for example).

To provide some background, a standard WIF encoded key is defined as the following, encoded in base58check.

```text
network version byte + private key + optional 0x01 byte denoting a compressed pubkey
```

_Extended_ WIF for the purposes here is defined as the following data structure, encoded in base58check.

```text
(network version byte + address type offset) + private key + (data)
```

To summarize, extended WIF allows us to provide extra information on the specific address type the private key shall correspond to, and the arbitrary data needed to spend it, while being backwards compatible with standard WIF.

The `bits wif` command makes this encoding task easier for us. To encode the private key associated with the sender address used above, we may use the following, for example.

Assuming the private key is stored in hexadecimal string format in a file named `sender_addr.hex`, we have,

```bash
bits wif -i sender_addr.hex -T p2wpkh -P
```

Output:

```bash
94oeEPncUHF7hrXPx7bdbSPrar8tx8bUFNihAg6qr6NAN8Txxrb
```

We supplied the `-T` option (`--addr-type`) to denote a `p2wpkh` address type and omitted the `-D` option (`--data`) for optional data, since a compressed pubkey is implied for SegWit p2wpkh addresses. Information and help on the necessary data as well as the various supported address types for extended WIF, can be seen with the `bits wif -h` command.

We can also review the encoding, by performing a _decode_ operation with the following.

```bash
echo -n 94oeEPncUHF7hrXPx7bdbSPrar8tx8bUFNihAg6qr6NAN8Txxrb | bits wif --decode | jq .
```

Output:

```bash
{
  "version": "f0",
  "network": "testnet",
  "addr_type": "p2wpkh",
  "key": "a7fb90f08460ae7cac8ddb29df999e47e942809a48f353d5fcfc84845d759904",
  "data": ""
}
```

Here we see the decoded address as a JSON structure. We see a version of hex `f0` indicating the `ef` testnet/regtest network version byte plus address offset of `01` for a P2WPKH address. We also see that information being provided by the `network` and `addr_type` keys, respectively. Finally we see the decoded private key value in hexadecimal string format, as well as the empty data key value, since no further data was appended.

Now, we can take this key back to our `bits send` command we used previously to sign the transaction for spenditure from the sender address of `bcrt1quzz3xxt2p0488hvlfelj78dq2pgcsjkd3auhwz`.

### Signing the transaction with `bits send`

Now, we simply take the command we used previously, but also leverage the `--sighash` option to specify the SIGHASH flag and denote a signing operation is to occur, and provide the WIF key(s) necessary for spenditure via the input, i.e.

```bash
echo -n 94oeEPncUHF7hrXPx7bdbSPrar8tx8bUFNihAg6qr6NAN8Txxrb | bits send bcrt1quzz3xxt2p0488hvlfelj78dq2pgcsjkd3auhwz bcrt1qddfhzmx80snsgudw8hr2qj9v3yz0vgd9wj5wca --send-fraction 0.5 --change-addr mp6nDfuUJWV1pNdcxNQtbU29eg3fJ6eK4c --sighash all
```

Output:

```bash
0100000000010189592b942d94a2265c4c986560363b5cf672fd9c6b8b5acc212759e436efa6150000000000ffffffff02a82b5009000000001600146b53716cc77c270471ae3dc6a048ac8904f621a5902f5009000000001976a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac0247304402202f14ed3b68d8a97faf5c7f176f55d8610d27adf85927d213de88f562d821d2eb02203c04f9b9280038f34de89990852c3b54b94174154be5b01621e9dc5a5c4488d201210365a02c1e0ca4e10aa8ff5283ac0476077c503a440699212744ded7235ba5f87500000000
```

Before sending, we can inspect the transaction with `bits tx --decode`, i.e.

```bash
echo 0100000000010189592b942d94a2265c4c986560363b5cf672fd9c6b8b5acc212759e436efa6150000000000ffffffff02a82b5009000000001600146b53716cc77c270471ae3dc6a048ac8904f621a5902f5009000000001976a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac0247304402202f14ed3b68d8a97faf5c7f176f55d8610d27adf85927d213de88f562d821d2eb02203c04f9b9280038f34de89990852c3b54b94174154be5b01621e9dc5a5c4488d201210365a02c1e0ca4e10aa8ff5283ac0476077c503a440699212744ded7235ba5f87500000000 | bits tx --decode | jq .
```

Output:

```bash
{
  "txid": "1053c3fe474b13156ad6611d5ebfaf6c500711f7742d5978ad3559d2b8beeeb3",
  "wtxid": "30fee44633756231d2bf767b91ca6ad1109538aa5dbbb5e6f3b52cc0dac3aa32",
  "version": 1,
  "txins": [
    {
      "txid": "89592b942d94a2265c4c986560363b5cf672fd9c6b8b5acc212759e436efa615",
      "vout": 0,
      "scriptsig": "",
      "sequence": "ffffffff"
    }
  ],
  "txouts": [
    {
      "value": 156249000,
      "scriptpubkey": "00146b53716cc77c270471ae3dc6a048ac8904f621a5"
    },
    {
      "value": 156250000,
      "scriptpubkey": "76a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac"
    }
  ],
  "witnesses": [
    [
      "304402202f14ed3b68d8a97faf5c7f176f55d8610d27adf85927d213de88f562d821d2eb02203c04f9b9280038f34de89990852c3b54b94174154be5b01621e9dc5a5c4488d201",
      "0365a02c1e0ca4e10aa8ff5283ac0476077c503a440699212744ded7235ba5f875"
    ]
  ],
  "locktime": 0
}
```

We now see the necessary signature data via the witness. Finally, we can again leverage our [configured](#configuring-bits-with-a-local-bitcoin-core-node) [Bitcoin Core](https://github.com/bitcoin/bitcoin) RPC node to send the raw transaction, e.g.

```bash
bits rpc sendrawtransaction "0100000000010189592b942d94a2265c4c986560363b5cf672fd9c6b8b5acc212759e436efa6150000000000ffffffff02a82b5009000000001600146b53716cc77c270471ae3dc6a048ac8904f621a5902f5009000000001976a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac0247304402202f14ed3b68d8a97faf5c7f176f55d8610d27adf85927d213de88f562d821d2eb02203c04f9b9280038f34de89990852c3b54b94174154be5b01621e9dc5a5c4488d201210365a02c1e0ca4e10aa8ff5283ac0476077c503a440699212744ded7235ba5f87500000000"
```

Output:

```bash
b3eebeb8d25935ad78592d74f71107506cafbf5e1d61d66a15134b47fec35310
```

Again, remember that this transaction must be mined in a block before we can view the fund transfer on the blockchain, (which offers us a nice segway to some of the last subcommands we will demo in this blog) but it should now be clear how `bits send` can provide an easier and more convenient method for sending funds.

### Mining blocks

As stated just above, the raw transaction we just sent via our [configured](#configuring-bits-with-a-local-bitcoin-core-node) [Bitcoin Core](https://github.com/bitcoin/bitcoin) RPC connection won't be reflected in the blockchain until it is mined. For mainnet, this will happen by some miner on the network at an average rate of every 10 minutes. However, the [bits](https://github.com/jtraub91/bits) CLI actually provides the `bits mine` command to mine blocks, which also depends on a locally configured Bitcoin Core node. This may take a _long_ time in practice for mainnet, but for our demonstration purposes on _regtest_, this command can be used to mine blocks and receive block rewards quickly. For example, to include the raw transaction we sent above (which is now part of the mempool) in a block, and receive the reward to a new address, e.g. `mvbD8F7cuoQcUcTfnLU72rMtB35P6wAHBJ`, we can use the following.

```bash
bits mine --limit 1 --recv-addr mvbD8F7cuoQcUcTfnLU72rMtB35P6wAHBJ
```

Output:

```bash
1 blocks mined. Reward sent to mvbD8F7cuoQcUcTfnLU72rMtB35P6wAHBJ
```

The `--limit` option is used to only mine the indicated number of blocks. If this option is omitted the `bits mine` command will mine indefinitely. Also, `--recv-addr` is used to indicate the address that the block reward shall be sent to.

Now with the previous transaction mined in a block, the following command can show us the respective amounts now allocated to the various addresses used in this demonstration.

```bash
bits rpc scantxoutset start '["addr(bcrt1quzz3xxt2p0488hvlfelj78dq2pgcsjkd3auhwz)", "addr(bcrt1qddfhzmx80snsgudw8hr2qj9v3yz0vgd9wj5wca)", "addr(mp6nDfuUJWV1pNdcxNQtbU29eg3fJ6eK4c)", "addr(mvbD8F7cuoQcUcTfnLU72rMtB35P6wAHBJ)"]' | jq .
```

Output:

```bash
{
  "success": true,
  "txouts": 830,
  "height": 828,
  "bestblock": "1f0f9cfb4360b649dbe4581e369dd0b80e3e7177a560f4f73e5c3873ba6f996f",
  "unspents": [
    {
      "txid": "b3eebeb8d25935ad78592d74f71107506cafbf5e1d61d66a15134b47fec35310",
      "vout": 0,
      "scriptPubKey": "00146b53716cc77c270471ae3dc6a048ac8904f621a5",
      "desc": "addr(bcrt1qddfhzmx80snsgudw8hr2qj9v3yz0vgd9wj5wca)#qq7tl9lw",
      "amount": 1.56249,
      "height": 828
    },
    {
      "txid": "b3eebeb8d25935ad78592d74f71107506cafbf5e1d61d66a15134b47fec35310",
      "vout": 1,
      "scriptPubKey": "76a9145e26bd0264f1a2451c552e313ad94b7127c7a76488ac",
      "desc": "addr(mp6nDfuUJWV1pNdcxNQtbU29eg3fJ6eK4c)#nxeam3zm",
      "amount": 1.5625,
      "height": 828
    },
    {
      "txid": "53b49c2022496267c0f962422b2a4fc27d09f88472a5536dd5640a36a2e5df63",
      "vout": 0,
      "scriptPubKey": "76a914a557ee47fee5eb85822017b26c03b9d1884a19e088ac",
      "desc": "addr(mvbD8F7cuoQcUcTfnLU72rMtB35P6wAHBJ)#4j85pt9g",
      "amount": 1.5625,
      "height": 828
    }
  ],
  "total_amount": 4.68749
}
```

Note that per consensus rules, the block reward will remain unavailable for spenditure for 100 blocks; so to make the funds now associated with `mvbD8F7cuoQcUcTfnLU72rMtB35P6wAHBJ` immediately available, we can mine 100 blocks, e.g.

```bash
bits mine --limit 100 --recv-addr ""
```

Output:

```bash
100 blocks mined. Reward sent to
```

Note that we provided an empty `--recv-addr` for demo purposes since we do not care about the block reward in this instance on regtest. This is not recommended for production purposes on mainnet! 🤑

### Closing bits

That about wraps it up for this blog post, but before we go, there are a couple more subcommands I'd like to explain.

Though unsupported (nor functionl nor advised 😅), there does exist a `bits p2p` command , which intends to eventually implement a functional native full node for [bits](https://github.com/jtraub91/bits), which would thereby deprecate support for [configuring](#configuring-bits-with-a-local-bitcoin-core-node) a [Bitcoin Core](https://github.com/bitcoin/bitcoin) RPC connection. Exposing this subcommand is only to show users what is in the codebase today, and to help to illustrate what is planned for future. Play around or hack on it, but don't expect it to do anything too useful at present.

Finally, the last subcommand I will mention is the `bits blockchain` command. Again, this command is planned to be able to explore and retrieve blockchain data, and will depend on a functioning `bits p2p` full node. In the meantime it supports _decoding_ raw blocks (via `bits blockchain --decode`), which can be useful for debugging. Other than that, it currently supports retrieving only the hard-coded genesis block of the Bitcoin blockchain, at the block height of 0, e.g.

```bash
bits blockchain 0
```

Output:

```bash
01000000000000000000000000000000000000000000000000000000000000000000000001000000010000000000000000000000000000000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac0000000029ab5f49ffff001d1dac2b7c0101000000010000000000000000000000000000000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac00000000
```

Like many `bits` subcommands, the output can be specified as raw binary, for which may be interesting to view, as this output contains some notable historical information 😉.

```bash
bits blockchain 0 -0
```

That concludes this post! Thanks for bearing with me 🙏 I hope you find this cli tool and library to be useful. Please feel free to reach out for questions or to just say hi 👋 Till next time ✌️ 😎 ❤️

### Citations

1. [xxd man page](https://linux.die.net/man/1/xxd)
2. [SEC 2: Recommended Elliptic Curve Domain Parameters](https://www.secg.org/sec2-v2.pdf)
