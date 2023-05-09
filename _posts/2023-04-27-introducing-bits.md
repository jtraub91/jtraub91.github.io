---
layout: post
title:  Introducing bits
categories: bitcoin python cli
author: Jason Traub
---

Introducing [bits](https://github.com/jtraub91/bits), a free and open source cli tool and pure Python library that I've developed for Bitcoin.

This post will serve as informal documentation for some of what you can do with [bits](https://github.com/jtraub91/bits) today, and what is planned for the future.

## Installation

```bash
pip install bits
```

## Usage

[bits](https://github.com/jtraub91/bits) is pure Python, and thus, can be used as a library by calling its various functions and methods directly (with documentation coming soon 🚀); however, often it's convenient to simply use a command line interface (CLI) to common functionality. The [bits](https://github.com/jtraub91/bits) CLI provides many different methods which are explained in detail in the subsequent sections.

## Command Line Interface

### The `bits` base command

The `bits` command, by itself, with no subcommand present, is a command in its own rite which will "convert input bits to bytes". Sharing functionality with standard Linux utility `xxd`, it may be used similarly in some respects.

With `xxd`, we can convert a raw binary input to a hex string like so,

```bash
head -c 8 /dev/urandom | xxd -p
```

Similarly we can do the following with `bits`

```bash
head -c 8 /dev/urandom | bits -1 -0x
```

The `-1` indicates that the _input_ is raw bytes, while the `-0x` flag indicates the _output_ shall be a hexadecimal string. `bits` supports three formats for input / output, as applicable: raw bytes, binary string, and hexadecimal string.

A nuanced improvement `bits` has over `xxd` in this regard, is that we don't have to supply `-c` for longer inputs, for example

```bash
head -c 32 /dev/urandom | xxd -p -c 32
```

becomes simply

```bash
head -c 32 /dev/urandom | bits -1 
```

(Notice that `-0x` can be absent as hexadecimal string is the default IO format for most commands)

But also, the support for binary strings provides functionality that `xxd` does not (nor any other CLI tool that I am aware of), in that it can convert between binary string representation. For example, with `xxd` in `-bits` (`-b`) mode, we can do,

```bash
head -c 32 /dev/urandom | xxd -b
```

to get a binary string representation of the input; but it outputs it in octets, which may be inconvenient for further processing. Furthermore, the `-revert` (`-r`) flag "[does] not work with [`-bits`] mode".

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

The `bits key` command will generate a bitcoin private key, which is nothing but a 256 bit random value strictly less than `SECP256K1_N`.

This, you may notice, _can_ be generated with the base command, e.g.

```bash
head -c 32 /dev/urandom | bits -0
```

but instead of having to manually confirm the output is below `SECP256K1_N` you can use `bits key` instead for convenience.

Also, note that if a command has input or output, it will accept a `--input-file` (`-in`, `-i`) and/or `--out-file` (`-out`, `-o`) optional argument, defaulting to stdin or stdout, respectively. Thus, to conceal the private key from being exposed on the command line, you may use the following command, for example.

```bash
bits key -out secret.hex
```

(or standard shell redirection, e.g. `bits key > secret.hex`)

For compatibility with legacy tooling (if you're into that sort of thing), the `bits key` command supports PEM-encoded private key output, e.g.

```bash
bits key -0pem
```

Note that `bits` _does not_ provide support for PEM-encoded `input` on the CLI, so you will need to convert back to a supported format (raw, bin, hex) in order to use with other `bits` (sub)commands. This can be done with `openssl ec` or `openssl asn1parse`, for example.

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

Now, you may want to create a Bitcoin address from your key pair. The `bits addr` command provides a convenient method to handle the encoding for various address types. This command supports generating legacy and segwit addresses alike, but expects the payload to passed in as the argument. For example, for legacy pay to pubkey addresses (P2PKH), we'll need to hash the pubkey first with `hash160`. The [bits](https://github.com/jtraub91/bits) CLI fortunately provides the `ripemd160`, `sha256`, `hash160`, and `hash256` crypto hashing subcommands for convenience. Thus, to generate a legacy Bitcoin address from our pubkey, we may use the following, for example.

```bash
bits hash160 -in pubkey.hex | bits addr | awk '{print $1}'
```

The output is piped into `awk` above only for readability, i.e. to produce a newline on the shell output since `bits addr` will technically output raw bytes, with no newline appended; however, in these situations `bits` provides the `--print` (`-P`) flag to do the same, for convenience. E.g.

```bash
bits hash160 -in pubkey.hex | bits addr -P
```

`bits addr` accepts a `--type` (`-T`) flag for address type which defaults to "p2pkh" for pay-to-pubkey adresses, but may be specified to be "p2sh", [BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki) pay-to-script-hash addresses.

A testnet address may be generated instead by leveraging the `--network` (`-N`) flag. E.g.

```bash
bits hash160 -in pubkey.hex | bits addr -N testnet -P
```

Segwit addresses can be generated similarly, e.g.

```bash
bits hash160 -in pubkey.hex | bits addr --witness-version 0 -P
```

The presence of the `--witness-version` (`--wv`) flag indicates a segwit address and its value represents the version. When this flag is present the `--type` flag is ignored. Segwit addresses identify themselves as pay to witness key hash (P2WKH) or pay to witness script hash (P2WSH) by the size of their payload, thus the user is expected to form and provide the correct payload as necessary. More on this later.

The `bits addr` command isn't doing anything magical. It accepts a payload and optional network and address type (or witness version) arguments, prepends the correct version byte and encodes in base58check or bech32, as appropriate. This can also be done manually with other [bits](https://github.com/jtraub91/bits) cli subcommands as we will see in the next section.

### Encoding (and decoding) with `base58` and `bech32`

The [bits](https://github.com/jtraub91/bits) cli provides methods for encoding and decoding base58(check) and bech32.

Recreating what was performed above, by the `bits addr` command, but instead with `bits base58` and `bits bech32`, repectively, we could do the following, for example.

For legacy P2PKH addresses:

- Take your pubkey and hash it like before
- Pre-prend the version byte (`0x00` for mainnet)
- Encode with [base58check](https://en.bitcoin.it/wiki/Base58Check_encoding)

Doing this, all at once, on the command line, would look something like the following.

```bash
bits hash160 -in pubkey.hex | awk '{print "00"$1}' | bits base58 --check
```

The `--check` option is supplied so that the checksum is appended. `bits base58` also supports a `--decode` option for retrieving the original payload, e.g.

```bash
echo -n <base58-check-encoded-address> | bits base58 --check --decode
```

The `--check` flag with the `--decode` flag present will make `bits` ensure that the checksum is valid.

Similarly, we can reproduce segwit address we previously created with `bits addr`, instead with `bits bech32`.

```bash
bits hash160 -in pubkey.hex | bits bech32 --hrp bc --wv 0 -P
```

As we can see `bits bech32` just gives a little more flexibility regarding the bech32 specification, whereas `bits addr` provides practical convenience. With `bits bech32` we can supply an arbitrary human readable part via `--hrp` and an optional Segwit version via the `--witness-version` (`--wv`) flag. From there, the data passed as input is encoded, with checksum appended. Conversely, the convenience of `bits addr` is that the address encoding is inferred from the options provided, i.e. `--network` (`-N`), `--type` (`-T`), and `--witness-version` (`--wv`).

Likewise, we can also _decode_ bech32, e.g.

```bash
echo -n <bech32-encoded> | bits bech32 --decode
```

If the encoded string is determined to be segwit address, the return of the previous command will be JSON output with keys "network", "witness_version", and "witness_program", otherwise the keys will be "hrp" and "payload".

### Mnemonic seeds and HD wallet operations

`bits` provides two subcommands for mnemonic / seed generation and hierarchical deterministic wallets per BIP32, BIP39, BIP43, and BIP44: `bits mnemonic` and `bits hd`.

To generate a mnemonic seed phrase you may simply use the following.

```bash
bits mnemonic
```

This command also supports providing your own entropy. For example,

```bash
head -c 32 /dev/urandom | bits mnemonic --from-entropy
```

Furthermore, we can provide the generated mnemonic as input to the `bits mnemonic` command, with the `--to-seed` or `--to-master-key` options present to generate the seed or master key, respectively. These options will cause `bits mnemonic` to prompt for a passphrase at the CLI (which can be blank).

```bash
echo <mnemonic-phrase> | bits mnemonic --to-master-key
passphrase: 
```

The base58check-encoded master key, provided as output of the previous command, is needed for further derivation per the `bits hd` command.

With `bits hd` you may derive private extended private keys (xprv) or extended public keys (xpub) by supplying the root key and a derivation path per BIP32, BIP43, and/or BIP44. For example,

```bash
echo -n <root-key> | bits hd "m/44'/0'/0'/0/0"
```

The leading `m` indicates private key derivation, implying a xprv must be supplied as input. The command will then derive and return the xprv at the derivation path provided, _but_ the xpub can be returned instead by supplying the `--xpub` flag. E.g.

```bash
echo -n <root-key> | bits hd "m/44'/0'/0'/0/0" --xpub
```

Also, a deserialized key can be emitted in JSON format (with keys "version", "depth", "parent_key_fingerprint", "child_no", and "key") via stderr by supplying the optional `--dump` flag.

Public derivation is supported and can be used by indicating a capital `M` in the derivation path. E.g.

```bash
echo -n <xpub> | bits hd "M/0" 
```

Also, of note, is that you aren't required to use the root master key as input for the derivation. Any derived key can be used as the starting point and derivation will happen relative to it. This may be useful so as to not expose the root master key on the same machine derivation is performed.

### Leveraging a Bitcoin Core Node

So far we've seen that the [bits](https://github.com/jtraub91/bits) CLI is able to do some basic bits to bytes manipulations and provide some utilites for generating and managing Bitcoin keys, addresses, etc. What [bits](https://github.com/jtraub91/bits) does not do currently, is provide a full node for consensus. For this, I recommend having a local Bitcoin Core node running (i.e. `bitcoind`) so that blockchain data can be downloaded, transactions can be relayed, and a node for verifying and maintaining consensus can be employed, strengthening the Bitcoin network.

Part of the plan for [bits](https://github.com/jtraub91/bits) is to implement a full node, with built-in rpc server, by version 1. There does actually exist code in the codebase that has started to implement such, but is not currently very functional nor supported.

In the meantime, I recommend leveraging Bitcoin Core here. As we continue thru this blog, you may want a Bitcoin Core node while using subcommands such as `bits tx` for looking up necessary blockchain data, but additionally and moreover, `bits` can be configured to directly interact with Bitcoin Core over rpc, either by sending raw RPC commands via the `bits rpc` subcommand, or by leveraging it internally, as with the `bits send` subcommand, for example. These will be explained, in depth, in the following sections.

To learn how to install Bitcoin Core on your OS, visit [bitcoin.org](https://bitcoin.org).

### Configuring `bits` with a local Bitcoin Core node

Nominal bits configuration is located in your user's home directory, in a directory named `.bits`, in a file named either `config.toml` or `config.json`, depending on support.

You may configure a connection via RPC with either cookie based authentication or a user / password. To connect via cookie based authentication, all you should configure is `rpc_url` and `rpc_datadir`. These may be specified at the CLI for dependent subcommands, or configured in `~/.bits/config.toml`, for example.

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
bits rpc help
```

### Creating raw transactions

We can manually create raw bitcoin transactions with bits using `bits tx`. We'll use an RPC connection to a local Bitcoin Core full node to get the necessary utxo information.
