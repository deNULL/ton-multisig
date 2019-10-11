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
`init.fif`

## Adding more keys to a wallet initialisation request
`add-keys.fif`

## Creating an order for a new money transfer
`new-order.fif`

## Adding signatures to a newly created order
`sign-order.fif`

## Sealing an order before upload
`seal-order.fif`

## Creating (and signing) a copy of previously uploaded (pending) order
`sign-sent-order.fif`

## Merging two copies of the same order
`merge-orders.fif`

## Inspecting an order before upload
`show-order.fif`

## Inspecting wallet's state
`show-state.fif`

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

# DRAFT BELOW

Сборка с геттерами:

func -o"my/code.fif" -P smartcont/stdlib.fc my/code.fc my/code-getters.fc

Без:

func -o"my/code.fif" -P smartcont/stdlib.fc my/code.fc

Баги/улучшения:

- не хватает метода udict_delete? (по аналогии с idict_delete?)

- зависание при использовании метода unpack_request

- ошибка с UNTILEND (в коде garbage collector'а)

- отказ от get-методов в пользу работы с данными из фифта (экономия места)

- было бы хорошо иметь механизм управления ассемблерным кодом из фифта (типа #ifdef), чтобы корректировать генерируемый код

- полезно иметь слово для вычисления id get-метода по его имени
(#ifdef в FunC -> #ifdef в асме -> #define в Fift -> в зависимости от наличия либо выкидывать кусок кода, либо нет)

- B@ выбрасывает Bytes?

Создание:

init.fif - создаем кошелек с n ключами, k требуется


append-keys.fif - добавляем ключ в код создания кошелька (TODO)





Apart from this basic functionality of the smart contract, you have to implement 

= get-methods that list all pending (partially signed) orders
orders

= get-methods that list all pending orders signed or not signed by a particular public key.
signedBy


Your interface Fift scripts should help the user:

= to create (BoC files with serialized) external messages with completely new orders (similarly to `wallet.fif` used by the simple wallet smart contract)

multisig-new-order.fif - создаем запрос на задание


= to extract and show the internal message (especially its destination address and value) and the list of signatures from a previously serialized external messages (loaded from a file)

multisig-show-order.fif - показать данные в запросе


= to add new signatures to such external messages using a local private key file (so that the holder of one private key might create an external message and send it by e-mail to the holder of another private key, who could add the second signature to the next holder and so on until the necessary amount of signatures is collected offline)

multisig-sign-order.fif - добавляем подпись в запрос


= to merge two external messages with the same body, but with different signature sets, into one external message with the union of these signature sets,

multisig-merge-orders.fif - объединить два запроса на одинаковое задание


= to create and sign a "new order" external message corresponding to a partially signed order recovered from the current state of the blockchain using one of the get-methods indicated above.

multisig-sign-sent-order.fif - импортируем существующее задание и делаем запрос на добавление подписи в него




Структура data:

32 бита: seqno
16 бит: pending, число запросов, ожидающих подписей
8 бит: n, число ключей в контракте
8 бит: k, число ключей, необходимых для отправки запроса

dict (публичный ключ => его 8 бит индекс)
dict (хэш запроса => инфо)
  для каждого запроса:
  8 бит: s, число подписей
  ref: тело запроса + mode (как в обычном кошельке) + expire_at
  dict (8 бит индекс ключа => 512-битная подпись)


Issues:
как использовать Bytes в кач-ве ключей dict?
как использовать Integer в кач-ве значений?


Сообщение:

512 бит подпись
32 бита: seqno
32 бита: valid_until
256 бит пуб ключ того, кто подписал _сообщение_ <- по факту тут будет ключ того, кто последним добавлял свою подпись
ref: тело запроса + mode (как в обычном кошельке) + expire_at
dict: 256-бит пубключ => 512-бит подпись запроса



Потрачено на инициализацию (5/3):
20000000000 - 19984905999 = 15094001 nano = 0.015 Gram


base_wallet: 108.481639584 -> 108.581536356 -> 108.704436350

Варианты мультисига:

1. статичный код vs обновляемый

По дефолту механизм обновления кода не предусмотрен. Альтернатива - можно сделать возможность обновления кода, тогда право это делать дается одному ключу (первому?). Либо делать это таким же голосованием. (Пока что без обновления)

2. с подписью всего сообщения или без

Вариант без подписи всего сообщения - это если использовать один seqno, но тогда нельзя параллельно собирать подписи на два разных запроса.
Поэтому пилим двойную подпись - сначала тело самого запрос (с его seqno), потом сообщение целиком (с текущим seqno).
У последнего будет две подписи, поэтому есть спецскрипт для отрезания лишней.

3. с get-методами или без

get-методы не очень полезны, если код статичный и обработка ответа все равно требует вызова fift-скриптов. Поэтому по умолчанию fift скрипты просто импортируют multisig-code.fif и с помощью него декодируют сериализированные данные.

Для заливки в блокчейн по умолчанию юзается multisig-code-nogetters.fc, который покороче.



Тесткейсы:

Создать кошелек 5/4

Проверить через show-order

Отправить в сеть

Создать запрос 0 с подписями 1 3 5 и коротким временем жизни, проверить

Создать запрос 1 с подписями 2 3 4 5, проверить

Отправить 1 в сеть

Запаковать запрос 0 с ключом 1 и новым seqno

Отправить 0 в сеть

Проверить, что запрос выполнился, деньги переведены

Создать запрос 2 с подписью 4, проверить

Отправить 2 в сеть

Проверить, что запрос попал в pending, в pending с 4 подписью, в pending без 5 подписи

Создать запрос 3 с подписями 1 2, проверить

Добавить подпись 3 в запрос 3, проверить

Отправить 3 в сеть

Проверить, что оба запроса в pending

Создать копию запроса 2 с подписями 4 и 5, проверить

Отправить копию в сеть

Проверить, что в запросе 2 две подписи

Создать новую копию запроса 2 с подписями 1, 4 и 2, проверить

Объединить новую копию и исходный запрос, проверить

Запаковать объединение 2 ключом, проверить

Отправить объединение в сеть

Проверить, что 2 запрос выполнился

Добавить 1 подпись в исходный 3 запрос

Отправить

Проверить, что счетчик не поменялся

Добавить 5 подпись к предыдущему запросу

Отправить

Проверить, что деньги ушли