# Description

This is a smart-contract for TON blockchain implementing basic multi-signature wallet. It was made by Denis Olshin as part of Telegram contest announced on 09/24/2019 (https://t.me/contest/102).

Instructions below assume that you are using TON's lite-client with FunC and Fift binaries available at PATH, and that you're familiar with those tools.

For details about building lite-client, please refer to https://github.com/ton-blockchain/ton/tree/master/lite-client-docs. For basic info about running Fift scripts and uploading messages to TON, please refer to https://github.com/ton-blockchain/ton/blob/master/doc/LiteClient-HOWTO.

# What's included

This directory contains following files:

* `common-utils.fif`
   A Fift library with some helper functions, that could be useful for creating any kind of smart contract. You don't need to run this file.
* `multisig-utils.fif`
   Similarly to `common-utils.fif`, this is a library file. However, it contains only functions specific to this particular multi-sig wallet implementation. Each other Fift script here includes it. You don't need to run this file either.
* `code.fc`
   Code of this smart contract, written in FunC. Note that it does not include get-methods (except for `seqno` method).
* `getters.fc`
   Get-methods of this smart contract. They are stored separately so you can upload your wallet code without them (it will still be functional, but will take less space).
* `code.fif`
   Compiled version of `code.fc`. Below you'll find instructions how to recompile it yourself.
* `code-getters.fif`
   Compiled version of `code.fc` + `getters.fc`.
* `init.fif`, `new-order.fif`, `sign-order.fif`, `sign-sent-order.fif`, `merge-orders.fif`, `add-keys.fif`, `seal-order.fif`, `show-order.fif`, `show-state.fif`, `purge-expired.fif`
   Fift scripts for creating new wallet, creating new money transfer requests, signing them and so on. Below you'll find detailed explanations about all of them.
* `test-init.fif`
   Fift script that simulates the initialisation of a wallet locally, without actually uploading it to blockchain.
* `test-message.fif`
   Fift script that simulates sending a message to a wallet locally. Loads the original contract state and returns the modified one.
* `test-method.fif`
   Fift script that simulates executing a get-method of a smart contract for a given state. It includes `code-getters.fif`, so it can call get-methods even if wallet was uploaded without them.

If you wish to make modifications to the wallet's code, it's better to test it using `test-...` scripts without actually uploading it to the blockchain. The same can be done in case something goes wrong (see "Troubleshooting" section below).

# (Re)building the wallet code

As was mentioned above, the smart contract code is located in `code.fc` and its getters are in `getters.fc`. These files are written in FunC language, so after you make any changes to them, you need to run FunC transpiler before you can upload the updated code.

Run these commands (`<path-to-source>` here is the root directory with the TON source code):

```
func -o"code-getters.fif" -P <path-to-source>/crypto/smartcont/stdlib.fc code.fc getters.fc
func -o"code.fif" -P <path-to-source>/crypto/smartcont/stdlib.fc code.fc
```

This should rebuild files `code-getters.fif` (full version of the wallet) and `code.fif` (stripped-down version, without getters). Now you can use `init.fif` to init your wallet (see "Initialising a new wallet" section below).

# How to use

Multi-signature wallet is a wallet managed by some number N of private keys (fixed at the moment of wallet creation). It accepts requests for money transfers (or any other internal message from this wallet), signed by any subset of those keys.

If a request was signed by M keys, where M > some number K (fixed at the moment of wallet creation as well), the corresponding message will be sent. Otherwise it will be stored as a pending request.

Owners of the wallet can add their signatures to pending requests at any moment.

Each request can optionally have expiry time set, so it will be void after that moment (even if missing signatures will arrive). Current implementation will keep it among pending requests until the next valid message will arrive.

The following sections will explain how to perform all possible action with your multi-sig wallet.

**NB**: Whenever you need to provide a file name as an argument to some Fift script, write it **without an extension**. The script will add the appropriate extension automatically. 

## Initialising a new wallet
`./init.fif <workchain-id> <n> [<k> <filename-base>] [-C <code-fif>]`

You need to specify workchain id for your new order, as well as N (total number of keys) and K (number of keys required to complete an order).

If K is omitted, it is set to N. You can set the name of your new wallet. The private keys will be stored in files `<wallet>-1.pk`, `<wallet>-2.pk` and so on. Using -C option you can choose the file with the smart contract's code (either 'code' or 'code-getters').

After running this command you should get a '`<wallet>-init-query.boc`' file. Upload it to TON (using `sendfile <wallet>-init-query.boc`) in your client to initialise your wallet (of course, you need to transfer some Grams to its address first).

## Adding more keys to a wallet initialisation request
`./add-keys.fif <wallet> <keyname-1> <keyname-2>... [-O <output-boc>] [-K <k>]`

Using this script you can add more owners **before initialisation of your smart contract**. You won't be able to change list of owners afterwards. Note that it will change the future address of the wallet (as it depends on the initial data). Private keys are loaded from (or stored to) files `<keyname-1>.pk`, `<keyname-2>.pk` and so on.

Option -k allows you to change the K param (number of keys required to complete an order).

## Creating an order for a new money transfer
`./new-order.fif <wallet> <dest-addr> <seqno> <amount> <keyname-1> <keyname-2>... [-B <body-boc>] [-O <output-boc>] [-E <seconds>]`

This script will generate a brand new order for money transfer from your multi-sig wallet. You can specify any number of private keys to sign it with.

After that, upload it using `sendfile <wallet>-query.boc` (instead of '`<wallet>-query`' you can specify custom name using an -O option).

Option -B allows you to load custom message body for your future transfer.

If you use option -E, your order will be valid only **for the next number of seconds you specified**. After that, it won't be completed even if missing signatures will arrive.

## Adding signatures to a newly created order
`./sign-order.fif <query-boc> <keyname-1> <keyname-2>... [-O <output-boc>] [-S <seqno>]`

Before uploading a new order, you can add more signatures to it using this script.

If you're adding signatures to a previously uploaded order, you should update the seqno to the actual value using the -s option. Note that this won't work for newly created order - if you change the seqno, all signatures stored in it previously will become invalid.

## Sealing an order before upload
`./seal-order.fif <query-boc> <keyname> [-O <output-boc>] [-S <seqno>]`

This script can be used as a final preparation before uploading an order to the network. It will reduce the size of the prepared message (by about 768 bits). Note that you should upload it right after that (if you add new signatures to it, you will lose the one that was stripped away).

Usage of this script is totally optional.

## Creating (and signing) a copy of previously uploaded (pending) order
`./sign-sent-order.fif <wallet> <seqno> <request-hash> <keyname-1> <keyname-2>... [-O <output-boc>]`

After an order is sent, it's checked by the smart contract. If it contains enough signatures, it will be executed right away, otherwise it will be stored in the list of pending orders.

This script allows you to prepare a new request for such pending order. First, you need to obtain it's hash — either via `show-order.fif` (before uploading it) or via `show-state.fif` (see below for details). Alternatively, if you uploaded the smart contract with the get-methods included, you can call `pending` or `pending_signed_by` to inspect the list of pending orders. The hash is the first number in the returned lists.

After a request to update an existing order is created, it can be used exactly like the original one — you can add signatures with `sign-order.fif`, merge it with its copy (or the original order) with `merge-orders.fif`, seal it with `seal-order.fif` or inspect its contents with `show-order.fif`.

## Merging two copies of the same order
`./merge-orders.fif <query1-boc> <query2-boc> <keyname> [-O <output-boc>] [-S <seqno>]`

If you happen to have two copies of the same order (signed by different parties), you can merge them together with this script. Of course, you need to be an owner to do that.

If you won't specify `seqno` with a -s option, it will be selected automatically as maximum seqno of the two request.

## Inspecting an order before upload
`./show-order.fif <query-boc>`

You can use this script before signing or uploading an order to validate its contents (message body, destination, amount of money to transfer, and list of already added signatures).

## Inspecting wallet's state
`./show-state.fif <data-boc>`

This script will help to examine the current state of the wallet. First, you need to download its state using the `saveaccountdata <filename> <addr>` command in the shell of your client. After that you can pass the generated boc-file to this script.

It should outputed detailed info about wallet's params, list of its owners, and list of currently pending orders. Alternatively, you can use get-methods to inspect those values (see the next section).

# Get methods

In case you've used the default (non-stripped down) version of the contract, it will include some get-methods. You can run them in the TON client using the `runmethod` command. Note that they return raw data, so you may prefer using `show-state.fif` instead (see "Inspecting wallet's state" section). 

List of available methods:
* `seqno`
   Returns the current stored value of seqno for this wallet. This method is available in the stripped-down version too.
* `pending_count`
   Returns current number of pending orders.
* `param_n`
   Returns number of owners of this wallet.
* `param_k`
   Returns number of signatures required to execute an order.
* `owners`
   Returns List consisting of Tuples, in which the first component is a public key (as a number) of an owner, and the second is its internal index.
* `pending`
   Returns List consisting of Tuples with following values: `(req_hash, req_seqno, req_expiry, req_mode, req_sigcount, req_signatures, req_message)`.`req_signatures` is a bit mask, where *i*th bit is set if this order collected the signature from the owner with that internal index.
* `pending_signed_by(pubkey, signed?)`
   Return List of pending orders, filtered by a pubkey (public key of an owner, passed as a number). If second param is `true`, returns only orders signed by that owner. Otherwise returns only orders NOT signed by him.

# Troubleshooting

After the request is uploaded to TON, there's no practical way to check what's happening with it (until it will be accepted). So if something goes wrong and your message is not accepted by the wallet, you can only guess why.

Fortunately, there's couple of scripts that will help in this situation. First, you need to perform `saveaccountdata <filename> <addr>` command in the TON client shell. This will produce a boc-file containing current state of your wallet.

Now you can inspect it using `show-state.fif`. Alternatively, you can manually call get-methods of the contract using `test-method.fif` (it should produce the same info, but in raw format).

But most importantly, you can run `test-message.fif` with a message file (that you were trying to upload) to simulate the execution of the smart contract, and check the TVM output. In addition to builtin errors, there are some error codes that could be thrown:

* Error **33**. *Invalid outer seqno*.
   The current stored seqno is different from the one in the incoming message. If this is a request to add signature(s) to an existing order, you can use `sign-order.fif` or `seal-order.fif` with a `-s <seqno>` option to fix the seqno. Otherwise (if this is a message to create a new order), you need to re-create it using `new-order.fif`.
* Error **34**. *Invalid outer signature*.
   The whole message has an incorrect signature.
* Error **35**. *Message is expired*.
   The whole message has a valid_until field set and it's in the past. Note that the provided Fift scripts do not set this field (you can set the expiration time for an order, but not for a message containing it).
* Error **36**. *Unknown signatory of the message*.
   Person who signed this message is not among owners of this wallet. 
* Error **37**. *Pending order not found*.
   You provided a hash of some order, but it was not found among pending orders. It was either expired or never uploaded at all.
* Error **38**. *Expired order*.
   You are trying to upload an order that has an expiration date in the past.
* Error **39**. *Invalid seqno in the new order*.
   Newly added orders must contain a seqno matching the current seqno of the wallet. You need to create a new order (with `new-order.fif`) with the actual seqno and sign it again.
* Error **40**. *Unknown signatory of the order*.
   Person who signed this order is not among owners of this wallet. 
* Error **41**. *Invalid order signature*.
   One of the order's signatures is incorrect.

# Fift words conventions

Fift language is quite flexible, but it can be difficult to read. There's two main reasons for that: stack juggling and no strict conventions for word names. To make the code more readable, some custom conventions were introduced within this repository:

`kebab-case-words()` are helper functions (defined in `common-utils.fif` or `multisig-utils.fif`). Note that the name includes the parentheses at the end. (The only exceptions are `maybe,` and `maybe@+`)

`CamelCaseWords` are constants, defined using a `=:` word.

Those styles are chosen to stand out from the builtin words and from each other as much as possible.

# Implementation details and thoughts on possible alternatives

As external messages by themselves are vulnerable for the replay attacks, some measures against them are required. For simple wallets it's done by introducing a seqno (sequence number) and/or valid_until field. Both of these fields are signed by the private key of wallet's owner.

For multi-signature wallets this mechanism becomes tricky, as there are multiple owners of the same wallet, and multiple signatures can be included in the same message. There are different ways to implement this signing process. Let's discuss pros and cons for some of them.

1. **Basic approach**. There's single seqno stored in smart contract's data, and a single seqno in the message. However, there's also a HashMap 256 (outside of the signed part of a message), mapping public keys of current signatories to their signatures.

   The benefit of this approach is that it's easy to implement.

   However, it has a noticable usability drawback. Imagine we create a message containing a new order. We include current seqno value in it, and then sign it with our private key, and send it (via email, for example) to other owners to sign. The problem is that now nobody of the owners should "touch" the wallet (which can be hard to enforce). Otherwise, any accepted message will increase the seqno and our signed message will become invalid. We would need to collect all signatures from the start.

2. **Individual sequence numbers**. Instead of single seqno, the wallet's storage includes separate numbers for each owner. At the moment some owner decides to sign a message, he appends his own seqno to it, and then generates the signature of the resulting cell (there's no need to send seqno explicitly, as it is stored within the wallet, and different seqno's will lead to different signatures).

   This approach is slightly better than the previous one: if one of the owners updates the state of the wallet, while signatures are still being collected, it does not void the whole message (it will only make his signature invalid, so it can be seen as a way to recall a vote).

   Unfortunately, it will still require for owners to cooperate on some level (if one includes his signature into a to-be-sent message, it prevents him from sending any other signed messages to this wallet). Additionally, it requires to store extra data for each owner.

3. **Double-signing approach**. What if we just exclude seqno from the signed part of a message? It makes collecting signatures very easy - as long as actual order does not change, each signature will remain valid. Of course, now we lost our protection against replay attack: anybody can just copy our signed message later, change seqno to the actual value, and re-send this order.

   We can fix that by adding one *outer* signature for the whole message (inluding the seqno). For that we can use any private key of the owners (and attach his public key to the message too). If other owner decides to attach more signatures, he'll just replace that public key with his own, and write his own signature for the resulting cell.

   The only problem is that now we are open for *internal* replay attacks. Outside attackers can not send duplicate orders, but any owner can (with other owners' signatures attached).

   But here we can notice that we only need to prevent adding new orders. As long as a message just adds signatures to an existing (pending) order, a duplicate won't change its state (as long as we are not counting duplicate signatures twice).

   To solve that, let's add an *inner* seqno to the message. It does not need to be always equal to the current seqno of the wallet - just at the moment when the order is added to the pending list.

   Now, if we want to collect multiple signatures via email (or other "slow process"), we can just add our own signature and upload the resulting order (we can think of this as a "create" call), which will freeze the inner seqno counter. After that we can start collecting signatures for that order (which can now be identified just by its hash).

   This is the approach I've chosen for this implementation. The another minor detail is that by default the last owner to sign an order attaches his signature two times, which is excessive. If we wish, we can remove the inner signature: the smart contract can consider the validity of the outer signature as the confirmation of our approval. After doing so we'll reduce the size of the message by about 768 bits, but other owners won't be able to add their inner signatures to it. (Refer to section "Sealing an order before upload" for details.)