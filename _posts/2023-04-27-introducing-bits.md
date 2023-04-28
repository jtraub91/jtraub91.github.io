---
layout: post
title:  "Introducing bits"
categories: bitcoin bits python cli
---

## Introduction

Over the past year, I've been working on [bits](https://github.com/jtraub91/bits), a cli tool and library for Bitcoin, which I've recently [released](https://github.com/jtraub91/bits/releases) under the MIT License. I started the project to learn and understand the Bitcoin protocol on a deeply technical level, but have steadily developed it into a useful tool for managing crypto and interacting with the Bitcoin network. Of note, `bits` is pure Python and has no 3rd party dependencies.

This post will serve as informal documentation for some of what you can do with `bits` today, and what is planned for the future.

### Why Python?

Python is not only my language of choice, but it's relatively easy to learn and the code can often be quite readable. In this respect, `bits` code intends to serve as a self-document for the Bitcoin protocol. The popularity of Python may also help to reach the widest audience of users and contributors possible.

### Why no dependencies?

In the spirit of learning and self-documenting, using a 3rd party package to perform some of the functionality feels like cutting corners. Also, by reducing the code surface area, and implementing each aspect directly, the goal is to achieve highly reliable code that users and developers can trust to manage their funds, minimizing the risk of a 3rd party package introducing an unforseen vulnerability.

## Installation

First of all, here's how to install it.

### Install from a release

Go to [releases](https://github.com/jtraub91/bits/releases) and download the latest copy. Then you can install it with `pip`, e.g.

```bash
pip install bits-0.2.0-py3-none-any.whl
```

### Intall from source

Or you can clone the repo and install from source. E.g.

```bash
git clone https://github.com/jtraub91/bits.git
cd bits
pip install .
```

### Install from pypi

`bits` is _not_ on pypi as of this post, but hopefully coming soon.

## Usage

As `bits` is Python, its various functions can be used directly (with documentation coming soon 🚀); however, often its very convenient to simply use a command line interface (CLI) to common functionality. The `bits` CLI provides many different functions which are explained in the following section.

## Command Line Interface

Built with Python's argparse, the `bits` CLI intends to provide all the common functionality most users will need, as well as serve as a self documenting structure of how its various Python functions can be put together.

Help, on the various CLI functions, can be viewed with the following command.

```bash
bits --help`
```

### The `bits` base command

The `bits` command, by itself, with no subcommand present, is a command in its own rite which will "convert input bits to bytes". Sharing functionality and inspired by standard *nix utility `xxd`, it can be used similarly in some respects.

With `xxd`, we can convert a raw binary input to a hex string like so,

```bash
head -c 8 /dev/urandom | xxd -p
```

Similarly we can do the following with `bits`

```bash
head -c 8 /dev/urandom | bits -1 -0x
```

The `-1` indicates that the _input_ is raw bytes, while the `-0x` flag indicates the _output_ shall be a hexadecimal string. `bits` supports three formats for input / output, as applicable: raw bytes, binary string, and hexadecimal string.

A couple nuanced improvements over `xxd` in this regard, is that we don't have to supply `-c` for longer inputs, for example

```bash
head -c 32 /dev/urandom | xxd -p -c 32
```

becomes simply

```bash
head -c 32 /dev/urandom | bits -1 
```

(Notice that `-0x` can be absent as hexadecimal string is the default IO format for most commands)

But also, the support for binary strings provides functionality that `xxd` does not (or any other cli tool that I am aware of), in that it can convert between binary string representation. For example, with `xxd` we can do,

```bash
head -c 32 /dev/urandom | xxd -bits
```

to get a binary string representation of the input, but it outputs it in octets, which may be inconvenient for piping to further processing. Furthermore, the `-revert` (`-r`) flag "[does] not work with [`-bits`] mode".

So with `xxd` we can convert raw bytes to a hexadecimal string, and revert.

```bash
head -c 32 /dev/urandom | xxd -p -c 32 | xxd -r -p
```

We cannot do the same with `-b`

```bash
head -c 32 /dev/urandom | xxd -b | xxd -r -b
xxd: sorry, cannot revert this type of hexdump
```

But, we can with `bits`

```bash
head -c 32 /dev/urandom | bits -1 -0b | bits -1b -0
```

### CLI subcommands

Now, let's get into more useful and specific Bitcoin functions. These are provided by various `bits` _subcommands_. Again, you can view a full list of available subcommands with `bits -h`. Subcommand specific help is also available with `bits <subcommand> -h`

The following command descriptions, will cover the functionality but also try to explain some aspects of Bitcoin you may be unaware of.

#### Generate a Bitcoin private key

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

(or standard shell redirection, e.g. `bits key > secret.hex`, of course)

For compatibility with legacy tooling (if you're into that sort of thing), the `bits key` command support PEM encoded private key output, e.g.

```bash
bits key -0pem
```

Note that `bits` _does not_ provide support for PEM encoded input on the CLI, so you will need to convert back to a supported format (raw, bin, hex) in order to use with other `bits` (sub)commands. This can be done with `openssl ec` or `openssl asn1parse` for example.

#### Create a public key from a private key

The `bits pubkey` command generates a public key (pubkey) from a private key. E.g.

```bash
bits pubkey -in secret.hex
```

And you can output a compressed pubkey with the `--compressed` (`-X`) flag.

```bash
bits pubkey -in secret.hex -X -out pubkey.hex
```

Notice we leverage the `-out` flag again above to save the pubkey to a file.

#### Creating a bitcoin address

Now, you may want to create a Bitcoin address from your key pair. The `bits addr` command provides a convenient method to handle the encoding for various address types. This command supports generating legacy and segwit addresses alike, but expects the payload to passed in as the argument. For example, for legacy pay to pubkey addresses (P2PKH), we'll need to hash the pubkey first with `hash160`. `bits` CLI fortunately provides the `ripemd160`, `sha256`, `hash160`, and `hash256` crypto hashing subcommands for convenience. Thus, to generate a legacy Bitcoin address from our pubkey, we may use the following, for example.

```bash
echo <pubkey> | bits hash160 | bits addr | awk '{print $1}'
```

(The output is piped into `awk` above only for readability, i.e. to produce a newline on the shell output since `bits addr` will technically output raw bytes)

Notice that we don't have to strip the trailing newline from the `echo <pubkey>` command since `bits` will do this automatically for binary string and hexadecimal string IO formats. Newlines are _not_ stripped for raw bytes IO format, however.

Alternatively, we can again leverage `-in` to read a pubkey from a previously saved file.

```bash
bits hash160 -in pubkey.hex | bits addr | awk '{print $1}'
```

For legacy addresses, `bits addr` accepts a `--type` (`-T`) flag for address type which can be p2pkh or p2sh, for pay-to-pubkey or pay-to-script-hash addresses, respectively (defaults to p2pkh). More on p2sh addresses later.

If you need to create a tesnet address instead, you can use the `--network` (`-N`) flag, e.g.

```bash
bits hash160 -in pubkey.hex | bits addr -N testnet | awk '{print $1}'
```

Segwit addresses can be generated similarly, e.g.

```bash
bits hash160 -in pubkey.hex | bits addr --witness-version 0 | awk '{print $1}'
```

The presence of the `--witness-version` flag indicates a segwit address and its value represents the version. When this flag is present the `--type` flag is ignored. Segwit addresses identify themselves as pay to witness key hash (P2WKH) or pay to witness script hash (P2WSH) by the size of their payload, thus the user is expected to form and provide the correct payload as necessary. More on this later.

The `bits addr` command isn't doing anything magical. It accepts a payload and optional network and address type (or witness version) arguments, and pre-pends the correct version byte and encodes in base58check or bech32, as appropriate. This can also be done manually with other bits cli subcommand as we will see in the next section.

#### Encoding (and decoding) with `base58` and `bech32`

The `bits` cli provides methods for encoding and decoding base58(check) and bech32.

