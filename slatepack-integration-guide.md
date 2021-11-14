- Title: slatepacks-integration-guide
- Authors: [Marek Narozniak](https://mareknarozniak.com) aka [renzokuken](https://forum.grin.mw/u/renzokuken/)
- Start date: Nov 14, 2021
---

# GRIN Integration Guide

This document describes how to implement slatepack workflow in the stack of a typical cryptocurrency exchange service. We assume the reader is already familiar with [GRIN digital cash project](https://grin.mw/) and is familiar with the notion of [slatepack](https://docs.grin.mw/wiki/transactions/slatepack/).

We intend to have this document as practical, technical and short as possible. Only GRIN-related notions are going to be explained here. We assume the reader is a technical person already familiar with concepts such as frontend, backend, daemon, process, API request *et cetera*.

## Setup

### Wallet listener

Install the most up to date implementation of [grin-wallet](https://github.com/mimblewimble/grin-wallet). Prepare the configuration that suits your needs (ports, GRIN node endpoint *et cetera*) then initialize wallet.

Run the wallet in both `owner_api` and `foreign_api` listener mode. For test purposes you can manually run it using the following command.

```
grin-wallet owner_api
```

and

```
grin-wallet listen
```

On production you will need to daemonize those two processes or at least launch them using a daemonized process manager.

### Communication with the Owner API

The `owner_api` endpoint provided by the wallet is not easy to reach. It requires you to [initialize the secure API](https://docs.rs/grin_wallet_api/3.1.0/grin_wallet_api/index.html#tymethod.init_secure_api). The process is tricky and inconvenient, but there are two implementations in

1. [Python](https://github.com/grinfans/grinmw.py)
2. [Node.js](https://gist.github.com/marekyggdrasil/82716db323b34019bf02237012e0d16f)

that you can either use directly either use as template for your own implementation.

Once you are able to initiate the secure API with the wallet instance exposing `owner_api` you interact with the wallet programmatically. That allows you to initiate transactions, receive them, generate or verify payment proofs *et cetera*.

The `foreign_api` does not require any additional steps to perform communication. It is a regular API.

### CRON job or updated

When wallet instance is left for too long without syncing it tends to become non-responsive. It is recommended to run a periodic job using CRON (or any scheduler of your preference) to regularly sync your wallet by making a request to [retrieve_summary_info](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.retrieve_summary_info) with `refresh_from_node` argument set to `TRUE`.

An alternative solution to that would be running the [updater](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.start_updater).

## Deposit

A high level description of the GRIN deposit process.

1. On the frontend display your wallets `slatepack` address (which can be provided using the [get_slatepack_address method](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.get_slatepack_address)) and a text input to the user with information to input the `slatepack` message there. User submits the data.
2. This time using the `foreign_api` provide the `slatepack` from the user using the [receive_tx method](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Foreign.html#method.receive_tx). In response you will get another `slatepack` which you will display to the user to finalize the transaction.

This is all you need to do to receive the payment in GRIN. In the process you will get the `tx_id` which you can use to identify the transaction. To check the status of it make sure you [open the wallet](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.open_wallet) and run [get_stored_tx](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.get_stored_tx).

## Withdrawal

A high level description of the GRIN withdrawal process.

1. Provide your user with an input method that allows to collect the `amount` and users `slatepack` address. Validate this data before proceeding.
2. In the backend [open the wallet](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.open_wallet) and run the [init_send_tx](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.init_send_tx) method. This will provide you with `slate` and output information.
3. Using [create_slatepack_message](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.create_slatepack_message) you can turn `slate` into the `slatepack` that is ready to be displayed to the user in your frontend. Next to the `slatepack` display a new text input and request user to provide the slatepack response.
4. Make sure you lock those outputs using [tx_lock_outputs](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.tx_lock_outputs) to avoid invalidating the transaction by accidental attempt of double spending.
5. When user submits new `slatepack` you may run the [finalize_tx](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.finalize_tx) method and post it to the network using [post_tx](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.post_tx) method.
6. Additionally, for your user convenience you might consider to generate the payment proof using [retrieve_payment_proof](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.retrieve_payment_proof) method.

This is the entire transaction flow. In case of mistakes at the step 3, you may cancel the transaction using the [cancel_tx](https://docs.rs/grin_wallet_api/4.0.0/grin_wallet_api/struct.Owner.html#method.cancel_tx) method.
