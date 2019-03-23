[back](./README.md)
# Channels - intended usage

## Introduction
You interact with an Aeternity node both through HTTP requests and WebSocket
connections.
To learn more about channels and their life cycle see [the doc](/channels/README.md).

In each channel there are two WebSocket client parties. For each channel, a new WebSocket connection is opened. Once the channel is opened - participants are
equal in every regard. They have different roles while opening and we have
names for them - `initiator` and `responder`. For short we will call them _the
parties_.

There are two basic types of interaction: persisted connection events and HTTP API calls.

### WebSocket life cycle
These are used for the scenario when all parties behave correctly and as
expected. The flow is the following:

1. [Channel open](#channel-open)
2. [Channel off-chain update](#channel-off-chain-update)
  * [Transfer](#transfer)
  * [Create a contract](#create-a-contract)
  * [Call a contract](#call-a-contract)
3. [Optionally leave/reestablish](#leave-reestablish)
4. [Channel mutual close](#channel-mutual-close)

There are a some WebSocket events that can occur while the connection is open
but are not necessarily part of the channel's life cycle.

* [Update error](#update-error)

* [Update conflict](#update-conflict)

* [Generic messages](#generic-messages)

* [Deposit](#deposit-events)

* [Withdrawal](#withdraw-events)

* [Getting state](#getting-state)

  * [Balances](#get-balances)

  * [Proof of inclusion](#get-proof-of-inclusion)

  * [Dry-run a contract](#dry-run-a-contract)

  * [Contract call](#get-contract-calls)

* [Cleaning local contract calls](#pruning-contract-calls)

Only steps 1 and 4 require chain interactions, step 2 and 3 are off-chain.

### HTTP requests
There are two types of HTTP requests:
* Total amount-modifying ones - [deposit](#deposit-transaction) and [withdrawal](#withdraw-transaction)
* [Channel-closing ones](#solo-closing-sequence) - [solo close](#solo-close-on-chain-transaction), [slash](#slash-on-chain-transaction) and [settle](#settle-on-chain-transaction)

## Channel open
In order to use a channel, it must be opened. Both parties negotiate parameters for the channel - for example the amounts to participate. Some of those are relevant to the chain and end up in a`channel_create_tx` that is posted on the chain. Once a certain amount of blocks have been mined on top of the one that included it, the channel is considered to be opened.

### Websocket protocol

The channel websocket api currently supports one protocol: [`json-rpc`](https://www.jsonrpc.org/specification). `legacy` protocol was removed. Choosen protocol has to be specified with the `protocol` option.

In the examples below, the `json-rpc` protocol is used.

Detailed message transcripts from test suites can also be found [for JSON-RPC](./examples/aehttp_integration_SUITE/json-rpc/).

### Channel parameters
Each channel has a set of parameters that is required for opening a
connection. Most of those are part of the `channel_create_tx` which is included
in the chain, and the others are metadata used for the connection itself. We
will describe them in two separate groups: one for the channel establishing
and another for optional timeouts.

  | Name | Type | Description | Required for open | Required/Used in reestablish | Part of the `channel_create_tx` |
  | ---- | ---- | ----------- | ----------------- | ---------------------------- | ------------------------------- |
  | initiator_id | string | initiator's public key | Yes | No | Yes |
  | responder_id | string | responder's public key | Yes | No | Yes |
  | lock_period | integer | amount of blocks for disputing a solo close | Yes | No | Yes |
  | push_amount | integer | initial deposit in favour of the responder by the initiator | Yes | No | No |
  | initiator_amount | integer | amount of tokens the initiator has committed to the channel | Yes | No | Yes |
  | responder_amount | integer | amount of tokens the responder has committed to the channel | Yes | No | Yes |
  | channel_reserve | integer | the minimum amount both peers need to maintain | Yes | No | Yes |
  | ttl | integer | minimum block height to include the `channel_create_tx` | No | No | Yes |
  | host | string | host of the `responder`'s node| Yes if `role=initiator` | No | No | No |
  | port | integer | the port of the `responder`s node| Yes if `role=initiator` | No | No | No |
  | role | string | the role of the client - either `initiator` or `responder` | Yes | Yes | No |
  | minimum_depth | integer | the minimum amount of blocks to be mined | No | No | No |

  `responder`'s port and host pair must be reachable from `initiator` network
  so unless participants are part of a LAN, they should be exposed to the
  internet as described [here](../../node/api/README.md).
  
  Once established, the channel follows a [predefined set of state
  transitions](/channels/README.md#overview). The implementation protects the
  client from edge cases when transitions take too long or never happen using
  a set of different timers - if the event doesn't occur in the specified time
  frame then the off-chain protocol is considered to be violated and the
  WebSocket connection is killed. Those are optionally configurable alongside
  with the channel establish settings. Keep in mind that those are only local
  values for the specific participant, protecting one's own interest. The two
  participants can have different timeout settings and still doing updates, as
  long as no timer fires.

  All timeout values are integers and represent the waiting time in
  milliseconds.

  | Name | Description | Default value |
  | ---- | ----------- | ------------- |
  | timeout_idle | the time waiting for a new event to be initiated | 600000 |
  | timeout_funding_create | the time waiting for the `initiator` to produce the create channel transaction after the `noise` session had been established | 120000 |
  | timeout_funding_sign | the time frame the other client has to sign an off-chain update after our client had initiated and signed it. This applies only for double signed on-chain intended updates: channel create transaction, deposit, withdrawal and etc. | 120000 |
  | timeout_funding_lock | the time frame the other client has to confirm an on-chain transaction reaching maturity (passing minimum depth) after the local node has detected this. This applies only for double signed on-chain intended updates: channel create transaction, deposit, withdrawal and etc. | 360000 |
  | timeout_sign | the time frame the client has to return a signed off-chain update or to decline it. This applies for all off-chain updates | 500000 |
  | timeout_accept | the time frame the other client has to react to an event. This applies for all off-chain updates that are not meant to land on-chain, as well as some special cases: opening a `noise` connection, mutual closing acknowledgement and reestablishing an existing channel | 120000 |
  | timeout_initialized | the time frame the responder has to accept an incoming noise session. Applicable only for initiator | timeout_accept's value |
  | timeout_awaiting_open | the time frame the initiator has to start an outgoing noise session to the responder's node. Applicable only for responder | timeout_idle's value |

  In the following examples we will be using the following parameters:

  | Name | Value |
  | ---- | ----- |
  | initiator_id | ak_8wWs1j2vhgjexQmKfgEBrG8ysAucRJdb3jsag3PJKjEeXswb7 |
  | responder_id | ak_bmtGbfP3SdPoJNZCQGjjzbKRje15J9CEcWYaL1gZyv2qEyiMe |
  | lock_period | 10 |
  | push_amount | 3 |
  | initiator_amount | 10 |
  | responder_amount | 10 |
  | channel_reserve | 2 |
  | ttl | 1000 |

  The `initiator` will be connecting to the `responder` on localhost:1234
  We will be using the tool [wscat](https://github.com/websockets/wscat)
  We assume the channel's WebSocket listener is set on port 3014 (default one)

### Responder WebSocket open
Using the set of prenegotiated parameters the responder connects
```bash
$ wscat --connect 'localhost:3014/channel?initiator_id=ak_8wWs1j2vhgjexQmKfgEBrG8ysAucRJdb3jsag3PJKjEeXswb7&responder_id=ak_bmtGbfP3SdPoJNZCQGjjzbKRje15J9CEcWYaL1gZyv2qEyiMe&lock_period=10&push_amount=3&initiator_amount=10&responder_amount=10&channel_reserve=2&ttl=1000&port=1234&role=responder&protocol=json-rpc'

connected (press CTRL+C to quit)
```

Note the `role=responder` as it is specific. Note also that the `host` is missing - it is not required for the responder.
The `port` being specified is the one the `responder`'s node will start
listening for the `initiator`'s connection. If the `responder`'s node is
behind a firewall or some port forwarding is done - this should be done before
the initiator starts connecting as it will fail.

At this point the `responder` is listening on address `0.0.0.0` for the `initiator`'s
connection on the specified port - `1234`.

### Initiator WebSocket open
Using the set of prenegotiated parameters the initiator connects
```bash
$ wscat --connect 'localhost:3014/channel?initiator_id=ak_8wWs1j2vhgjexQmKfgEBrG8ysAucRJdb3jsag3PJKjEeXswb7&responder_id=ak_bmtGbfP3SdPoJNZCQGjjzbKRje15J9CEcWYaL1gZyv2qEyiMe&lock_period=10&push_amount=3&initiator_amount=10&responder_amount=10&channel_reserve=2&ttl=1000&host=localhost&port=1234&role=initiator&protocol=json-rpc'

connected (press CTRL+C to quit)
```

Note the `role=initiator` as it is specific. Note also the `host` and `port`
being provided by the `responder`.

### Connection opened messages
Parties' WebSocket clients receive messages for the opening of the TCP
connection.

#### Initiator connection opened message
The initiator receives the following message
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": null,
    "data": {
      "event": "channel_accept"
    }
  },
  "version": 1
}
```

#### Responder connection opened message
The responder receives the following message
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": null,
    "data": {
      "event": "channel_open"
    }
  },
  "version": 1
}
```

### Create transaction signing
The `channel_create_tx` is sent subsequently to both parties and they co-sign
it. Then it is posted to the chain.

#### Initiator signs the tx
The initiator receives a message containing the unsigned transaction
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.initiator_sign",
  "params": {
    "channel_id": null,
    "data": {
      "tx": "tx_+IEyAaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGP6olImAAoQFnHFVGRklFdbK0lPZRaCFxBmPYSJPN0tI2A3pUwz7uhIYkYTnKgAACCgCGEjCc5UAAwKCjPk7CXWjSHTO8V2Y9WTad6D/5sB8yCR8WumWh0WxWvwacFM/z"
    }
  },
  "version": 1
}
```

Initiator is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.initiator_sign",
  "params": {
    "tx": "tx_+MsLAfhCuECaHxrOXQB62F7zLVyRAEpR/LxVwFqcn6XlWVmkxeD/IRO0N7AiA5MI5pJAlpcgoDWS6jMszatmzDyWm698OUcBuIP4gTIBoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZoY/qiUiYAChAWccVUZGSUV1srSU9lFoIXEGY9hIk83S0jYDelTDPu6EhiRhOcqAAAIKAIYSMJzlQADAoKM+TsJdaNIdM7xXZj1ZNp3oP/mwHzIJHxa6ZaHRbFa/BuQuHu8="
  }
}
```

Please note that the these are just example transactions and you're to have
different values for those.

#### Responder is informed
The responder receives the following message
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": null,
    "data": {
      "event": "funding_created"
    }
  },
  "version": 1
}
```

#### Responder signs the tx
After being informed for the initiator's signing the responder receives a message containing the unsigned transaction to be signed as well
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.responder_sign",
  "params": {
    "channel_id": null,
    "data": {
      "tx": "tx_+IEyAaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGP6olImAAoQFnHFVGRklFdbK0lPZRaCFxBmPYSJPN0tI2A3pUwz7uhIYkYTnKgAACCgCGEjCc5UAAwKCjPk7CXWjSHTO8V2Y9WTad6D/5sB8yCR8WumWh0WxWvwacFM/z"
    }
  },
  "version": 1
}
```

Note that this is the same transaction. Responder is to decode the transaction, inspect its contents, sign it, encode it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.responder_sign",
  "params": {
    "tx": "tx_+MsLAfhCuED03nq3Nu91Votu/GBda41x1yC826uIeN6rF4svrQgomkBhwUPtAXGuo9dwetzW7ku9Co6JQ0seGtzt4nzA/wsDuIP4gTIBoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZoY/qiUiYAChAWccVUZGSUV1srSU9lFoIXEGY9hIk83S0jYDelTDPu6EhiRhOcqAAAIKAIYSMJzlQADAoKM+TsJdaNIdM7xXZj1ZNp3oP/mwHzIJHxa6ZaHRbFa/BsMX/1Q="
  }
}
```

#### Initiator is informed
The initiator receives the following message
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": null,
    "data": {
      "event": "funding_signed"
    }
  },
  "version": 1
}
```

#### Signed channel_create_tx
Both participants receive the co-signed `channel_create_tx`:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.on_chain_tx",
  "params": {
    "channel_id": null,
    "data": {
      "tx": "tx_+QENCwH4hLhAmh8azl0Aethe8y1ckQBKUfy8VcBanJ+l5VlZpMXg/yETtDewIgOTCOaSQJaXIKA1kuozLM2rZsw8lpuvfDlHAbhA9N56tzbvdVaLbvxgXWuNcdcgvNuriHjeqxeLL60IKJpAYcFD7QFxrqPXcHrc1u5LvQqOiUNLHhrc7eJ8wP8LA7iD+IEyAaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGP6olImAAoQFnHFVGRklFdbK0lPZRaCFxBmPYSJPN0tI2A3pUwz7uhIYkYTnKgAACCgCGEjCc5UAAwKCjPk7CXWjSHTO8V2Y9WTad6D/5sB8yCR8WumWh0WxWvwZdPEBJ"
    }
  },
  "version": 1
}
```

Using its hash, participants can track its progress on the chain: entering the mempool, block inclusion and a number of confirmations.

#### Transaction in mempool
At this point both parties had received the co-signed the `channel_create_tx` transaction. The transaction is posted by the state
channel's software to the node and goes to the mempool. Having calculated its hash, one can validate it using the external HTTP API:
```bash
$ curl 'http://localhost:3013/v2/transactions/th_hNyHzj4dSzyBqReAMR36GGz1mhuXxQFuES3AnPkXkuY2w6dZb'
```
if the `block_hash` is `none` - then the transaction is still in the mempool.

### Block inclusion
When the transaction is included in a block - this is the first confirmation. A block height timer is
started and it ends after `minimum_depth + 1` confirmations. Default value for
it is 4, so 5 blocks need to be mined. As a result, each party will receive
two kinds of confirmation.

An update from one's own node that the block height needed is reached:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": null,
    "data": {
      "event": "own_funding_locked"
    }
  },
  "version": 1
}
```
An update from one's own node that the other party had confirmed that the block
height needed is reached:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "funding_locked"
    }
  },
  "version": 1
}
```

### Initial state
After both parties have confirmed that the funding is signed - they can proceed
with sending the messages for off-chain updates. The inital state is the one
described in the create transaction.

#### Open confirmation
After both parties have co-signed the state update both of them will receive a info for the channel open:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "open"
    }
  },
  "version": 1
}
```

From this point on, the channel is considered to be opened.

## Channel off-chain update
After the channel has been opened and before it has been closed there is a
channel state that is updated when needed. The updates are off-chain and
broadcasted only between parties in the channel. The state is a full state
tree that holds all the latest accounts, contracts and contract calls.
A state is considered to be valid only if both parties have agreed upon it.
Agreement it proven with signing a message that contains the channel id, round
and root of the state tree (state_hash). States are ordered by their round - the greater the round,
the newer the state. The latest channel state is the last valid state, having
the greatest round. At any time the latest state can be used for unilaterally closing the channel.

### Channel state
There are a couple of different types that could define the channel state.
Those are deposit, withdrawal and off-chain transactions. They all containt at
least the following data:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | channel id | string | ID of the channel|
  | state_hash | string | root of the state tree |
  | round | integer | current round |

You can find further information for them as it follows:

* [deposit transaction](../../serializations.md#channel-deposit-transaction)
* [withdrawal transaction](../../serializations.md#channel-withdraw-transaction)
* [off-chain transaction](../../serializations.md#channel-off-chain-transaction)

Each subsequent state has a `round` increased with 1

Since both participants are peers, they can both trigger new updates to the
state.
Since one of them starts the update and the other acknowledges is below we are
going to use `starter` and `acknowledger`. Both the initiator and the
responder can take either of the roles.

### Transfer
The transfer update is moving tokens from one channel account to another. The update is a change to be applied on top of the latest state. It has the
following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | from | string | sender's public key |
  | to | string | receiver's public key |
  | amount | integer | the amount given |

Sender and receiver are the channel parties. Both the initiator and responder
can take those roles. Any public key outside of the channel is considered invalid.

#### Start transfer update
##### Trigger a transfer update
The starter sends a message containing the desired change
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update.new",
  "params": {
    "amount": 1,
    "from": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
    "to": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC"
  }
}
```

The `starter` might take the role of `from` or `to` so the `starter` can
trigger sending or request for tokens.

##### Starter signs updated state
The starter receives a message containing the updated channel state as an
off-chain transaction
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update",
  "params": {
    "channel_id": "ch_2eZhEJ676pykdcHHcTZuSF346bq1ru3qnAiT2KowopwrPKwy7t",
    "data": {
      "tx": "tx_+JU5AaEG2P0xPT5jHllUSQG3v4pr0wwyyauU5up8JR2EHRpwfRoC+E24S/hJggI6AaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2ahAWccVUZGSUV1srSU9lFoIXEGY9hIk83S0jYDelTDPu6EAaDVKneBkzhWnEbC2q9FakzUh62kF75skKy/zAal9PdC4RgGEv0="
    }
  },
  "version": 1
}
```

The starter is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "tx": "tx_+N8LAfhCuEAJucXkQXN/tCSLeXjp9g5Em8OrdzF1kjm1UNLjyozxCdlzsizi0RLWYkjSmj3P7inUPM6Nvde4ZLJluwd3b7AOuJf4lTkBoQbY/TE9PmMeWVRJAbe/imvTDDLJq5Tm6nwlHYQdGnB9GgL4TbhL+EmCAjoBoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZqEBZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oQBoNUqd4GTOFacRsLar0VqTNSHraQXvmyQrL/MBqX090LhzxWeeQ=="
  }
}
```

#### Acknowledger update
The acknowledger receives an info message indicating an upcoming change:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2eZhEJ676pykdcHHcTZuSF346bq1ru3qnAiT2KowopwrPKwy7t",
    "data": {
      "event": "update"
    }
  },
  "version": 1
}
```

Then the acknowledger receives a new message containing the updated channel state as an off-chain transaction
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update_ack",
  "params": {
    "channel_id": "ch_2eZhEJ676pykdcHHcTZuSF346bq1ru3qnAiT2KowopwrPKwy7t",
    "data": {
      "tx": "tx_+JU5AaEG2P0xPT5jHllUSQG3v4pr0wwyyauU5up8JR2EHRpwfRoC+E24S/hJggI6AaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2ahAWccVUZGSUV1srSU9lFoIXEGY9hIk83S0jYDelTDPu6EAaDVKneBkzhWnEbC2q9FakzUh62kF75skKy/zAal9PdC4RgGEv0="
    }
  },
  "version": 1
}
```

Note that this is the same transaction as the one that the starter had already signed. The acknowledger is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update_ack",
  "params": {
    "tx": "tx_+N8LAfhCuED01ftyCc1MJl77Q9rkyqrVhfP/AAJG1+iXGW0OLSWB2g9ZJoV8X1j+b5wB/xpCYMdavI1AnIalKIqguQ+uQ9MMuJf4lTkBoQbY/TE9PmMeWVRJAbe/imvTDDLJq5Tm6nwlHYQdGnB9GgL4TbhL+EmCAjoBoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZqEBZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oQBoNUqd4GTOFacRsLar0VqTNSHraQXvmyQrL/MBqX090LhJL4cvw=="
  }
}
```

#### Finish update
After both the parties have signed the new updated state of the channel - it is
considered the latest one. Corresponding update messages are sent to both
parties to indicate it. The payload of the message contains the latest
co-signed off-chain update so the participants can persist it locally.
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "channel_id": "ch_2eZhEJ676pykdcHHcTZuSF346bq1ru3qnAiT2KowopwrPKwy7t",
    "data": {
      "state": "tx_+QEhCwH4hLhACbnF5EFzf7Qki3l46fYORJvDq3cxdZI5tVDS48qM8QnZc7Is4tES1mJI0po9z+4p1DzOjb3XuGSyZbsHd2+wDrhA9NX7cgnNTCZe+0Pa5Mqq1YXz/wACRtfolxltDi0lgdoPWSaFfF9Y/m+cAf8aQmDHWryNQJyGpSiKoLkPrkPTDLiX+JU5AaEG2P0xPT5jHllUSQG3v4pr0wwyyauU5up8JR2EHRpwfRoC+E24S/hJggI6AaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2ahAWccVUZGSUV1srSU9lFoIXEGY9hIk83S0jYDelTDPu6EAaDVKneBkzhWnEbC2q9FakzUh62kF75skKy/zAal9PdC4cw1EJo="
    }
  },
  "version": 1
}
```

After that a new state updated can be triggered.

### Create a contract
The create contract update is creating a contract inside the channel's internal state tree. The update is a change to be applied on top of the latest state. It has the
following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | vm_version | integer | version of the AEVM |
  | abi_version | integer | version of the ABI |
  | deposit | integer | initial amount the owner of the contract commits to it |
  | code | string | api encoded compiled AEVM byte code |
  | call_data | string | api encoded compiled AEVM call data for the code |

That would create a contract with the poster being the `owner` of it. Poster
commits initially a `deposit` amount of tokens to the new contract.

#### Start create contract update
##### Trigger a create contract update
The owner sends a message containing the desired change
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update.new_contract",
  "params": {
    "abi_version": 1,
    "call_data": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACC5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAnHQYrA==",
    "code": "cb_+QTcRgGgMGbgp7wD6B3+n/ohfx9EzDl65rI64xWTcLSo2cJ7otP5A2P5AcuguclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeqEaW5pdLhgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA///////////////////////////////////////////uQFAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAFAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQD//////////////////////////////////////////wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD///////////////////////////////////////////5AZKg7nfmdqoCk7VwhfBIleas1RJ1aXkZ3la8nAyrmjkRcHSLY2FuX3Jlc29sdmW5ASAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAP//////////////////////////////////////////AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAG4QAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC5AVBiAABkYgAAhJGAgIBRf7nJVvKLMUmp9Zh6pQXz2hsiCcxXOSNABiu2wb2fn5nqFGIAAURXUIBRf+535naqApO1cIXwSJXmrNUSdWl5Gd5WvJwMq5o5EXB0FGIAAK9XUGABGVEAW2AAGVlgIAGQgVJgIJADYAOBUpBZYABRWVJgAFJgAPNbYACAUmAA81tZWWAgAZCBUmAgkANgABlZYCABkIFSYCCQA2ADgVKBUpBWW2AgAVGAUZBgIAFRWVCBgZJQklBQYABgAH9hktnKsTLmK0AbjhU+HchD/pn4We3jfElNxXAcNfa1ZGABWZCBUllgYAGQgVJgIJADhIFSYCCQA4WBUmAgkANgyIFSYABgAFrxgFFgABRiAAEsV4BRYAEUYgABNldQYAEZUQBbUGAAW5FQUJBWW2AgAVFgAZBQYgABMFZbUFCCkVBQYgAAjFZiG02X",
    "deposit": 10,
    "vm_version": 3
  }
}
```

##### Owner signs updated state
The owner receives a message containing the updated channel state as an
off-chain transaction

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+QW+OQGhBsYNEx0Z+hJ2sqygIgO1x7QvI4bfS5Tu+TpqeLgHdvDVePkFdbkFcvkFb4ICPQGhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmgwMAAbkE3/kE3EYBoDBm4Ke8A+gd/p/6IX8fRMw5euayOuMVk3C0qNnCe6LT+QNj+QHLoLnJVvKLMUmp9Zh6pQXz2hsiCcxXOSNABiu2wb2fn5nqhGluaXS4YAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP//////////////////////////////////////////7kBQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEA//////////////////////////////////////////8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA///////////////////////////////////////////+QGSoO535naqApO1cIXwSJXmrNUSdWl5Gd5WvJwMq5o5EXB0i2Nhbl9yZXNvbHZluQEgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQD//////////////////////////////////////////wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABuEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAuQFQYgAAZGIAAISRgICAUX+5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6hRiAAFEV1CAUX/ud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdBRiAACvV1BgARlRAFtgABlZYCABkIFSYCCQA2ADgVKQWWAAUVlSYABSYADzW2AAgFJgAPNbWVlgIAGQgVJgIJADYAAZWWAgAZCBUmAgkANgA4FSgVKQVltgIAFRgFGQYCABUVlQgYGSUJJQUGAAYAB/YZLZyrEy5itAG44VPh3IQ/6Z+Fnt43xJTcVwHDX2tWRgAVmQgVJZYGABkIFSYCCQA4SBUmAgkAOFgVJgIJADYMiBUmAAYABa8YBRYAAUYgABLFeAUWABFGIAATZXUGABGVEAW1BgAFuRUFCQVltgIAFRYAGQUGIAATBWW1BQgpFQUGIAAIxWCrhgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACC5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoENFifQaMeViebmpNVYxb64/e6U6h2ZdFWxzDohnQi7ffhOmgQ=="
    }
  },
  "version": 1
}
```
The owner is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "tx": "tx_+QYKCwH4QrhACI1aho4Hr/Ebapb7yf3ItbxjRKC4HaNc/vIPeVocGMawlPjeW1puZLA5b/VxRtEBgFhfIy7HI1TbGzDOhS/xDrkFwfkFvjkBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1Xj5BXW5BXL5BW+CAj0BoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZoMDAAG5BN/5BNxGAaAwZuCnvAPoHf6f+iF/H0TMOXrmsjrjFZNwtKjZwnui0/kDY/kBy6C5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6oRpbml0uGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD//////////////////////////////////////////+5AUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAP//////////////////////////////////////////AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP///////////////////////////////////////////kBkqDud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdItjYW5fcmVzb2x2ZbkBIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEA//////////////////////////////////////////8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAbhAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALkBUGIAAGRiAACEkYCAgFF/uclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeoUYgABRFdQgFF/7nfmdqoCk7VwhfBIleas1RJ1aXkZ3la8nAyrmjkRcHQUYgAAr1dQYAEZUQBbYAAZWWAgAZCBUmAgkANgA4FSkFlgAFFZUmAAUmAA81tgAIBSYADzW1lZYCABkIFSYCCQA2AAGVlgIAGQgVJgIJADYAOBUoFSkFZbYCABUYBRkGAgAVFZUIGBklCSUFBgAGAAf2GS2cqxMuYrQBuOFT4dyEP+mfhZ7eN8SU3FcBw19rVkYAFZkIFSWWBgAZCBUmAgkAOEgVJgIJADhYFSYCCQA2DIgVJgAGAAWvGAUWAAFGIAASxXgFFgARRiAAE2V1BgARlRAFtQYABbkVBQkFZbYCABUWABkFBiAAEwVltQUIKRUFBiAACMVgq4YAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAguclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKBDRYn0GjHlYnm5qTVWMW+uP3ulOodmXRVscw6IZ0Iu30SifBA="
  }
}
```

#### Acknowledger update
The acknowledger receives an info message indicating an upcoming change:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "update"
    }
  },
  "version": 1
}
```

Then the acknowledger receives a new message containing the updated channel state as an
off-chain transaction
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update_ack",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+QW+OQGhBsYNEx0Z+hJ2sqygIgO1x7QvI4bfS5Tu+TpqeLgHdvDVePkFdbkFcvkFb4ICPQGhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmgwMAAbkE3/kE3EYBoDBm4Ke8A+gd/p/6IX8fRMw5euayOuMVk3C0qNnCe6LT+QNj+QHLoLnJVvKLMUmp9Zh6pQXz2hsiCcxXOSNABiu2wb2fn5nqhGluaXS4YAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP//////////////////////////////////////////7kBQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEA//////////////////////////////////////////8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA///////////////////////////////////////////+QGSoO535naqApO1cIXwSJXmrNUSdWl5Gd5WvJwMq5o5EXB0i2Nhbl9yZXNvbHZluQEgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQD//////////////////////////////////////////wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABuEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAuQFQYgAAZGIAAISRgICAUX+5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6hRiAAFEV1CAUX/ud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdBRiAACvV1BgARlRAFtgABlZYCABkIFSYCCQA2ADgVKQWWAAUVlSYABSYADzW2AAgFJgAPNbWVlgIAGQgVJgIJADYAAZWWAgAZCBUmAgkANgA4FSgVKQVltgIAFRgFGQYCABUVlQgYGSUJJQUGAAYAB/YZLZyrEy5itAG44VPh3IQ/6Z+Fnt43xJTcVwHDX2tWRgAVmQgVJZYGABkIFSYCCQA4SBUmAgkAOFgVJgIJADYMiBUmAAYABa8YBRYAAUYgABLFeAUWABFGIAATZXUGABGVEAW1BgAFuRUFCQVltgIAFRYAGQUGIAATBWW1BQgpFQUGIAAIxWCrhgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACC5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoENFifQaMeViebmpNVYxb64/e6U6h2ZdFWxzDohnQi7ffhOmgQ=="
    }
  },
  "version": 1
}
```

Note that this is the same transaction as the one that the owner had already signed. The acknowledger is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update_ack",
  "params": {
    "tx": "tx_+QYKCwH4QrhA0UyT1fuIP3p7HE07MSOLJa/h4kb9VVBRBuhrK/62wDl0qGkqSTDnC+cwmcxbyyVZnNfyN0zFWXyoTcFs+Q/7DrkFwfkFvjkBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1Xj5BXW5BXL5BW+CAj0BoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZoMDAAG5BN/5BNxGAaAwZuCnvAPoHf6f+iF/H0TMOXrmsjrjFZNwtKjZwnui0/kDY/kBy6C5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6oRpbml0uGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD//////////////////////////////////////////+5AUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAP//////////////////////////////////////////AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP///////////////////////////////////////////kBkqDud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdItjYW5fcmVzb2x2ZbkBIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEA//////////////////////////////////////////8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAbhAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALkBUGIAAGRiAACEkYCAgFF/uclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeoUYgABRFdQgFF/7nfmdqoCk7VwhfBIleas1RJ1aXkZ3la8nAyrmjkRcHQUYgAAr1dQYAEZUQBbYAAZWWAgAZCBUmAgkANgA4FSkFlgAFFZUmAAUmAA81tgAIBSYADzW1lZYCABkIFSYCCQA2AAGVlgIAGQgVJgIJADYAOBUoFSkFZbYCABUYBRkGAgAVFZUIGBklCSUFBgAGAAf2GS2cqxMuYrQBuOFT4dyEP+mfhZ7eN8SU3FcBw19rVkYAFZkIFSWWBgAZCBUmAgkAOEgVJgIJADhYFSYCCQA2DIgVJgAGAAWvGAUWAAFGIAASxXgFFgARRiAAE2V1BgARlRAFtQYABbkVBQkFZbYCABUWABkFBiAAEwVltQUIKRUFBiAACMVgq4YAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAguclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKBDRYn0GjHlYnm5qTVWMW+uP3ulOodmXRVscw6IZ0Iu3zYgFGM="
  }
}
```

#### Finish update
After both the parties have signed the new updated state of the channel - it is
considered the latest one. Corresponding update messages are sent to both
parties to indicate it. The payload of the message contains the latest
co-signed off-chain update so the participants can persist it locally.
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "state": "tx_+QZMCwH4hLhACI1aho4Hr/Ebapb7yf3ItbxjRKC4HaNc/vIPeVocGMawlPjeW1puZLA5b/VxRtEBgFhfIy7HI1TbGzDOhS/xDrhA0UyT1fuIP3p7HE07MSOLJa/h4kb9VVBRBuhrK/62wDl0qGkqSTDnC+cwmcxbyyVZnNfyN0zFWXyoTcFs+Q/7DrkFwfkFvjkBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1Xj5BXW5BXL5BW+CAj0BoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZoMDAAG5BN/5BNxGAaAwZuCnvAPoHf6f+iF/H0TMOXrmsjrjFZNwtKjZwnui0/kDY/kBy6C5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6oRpbml0uGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD//////////////////////////////////////////+5AUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAP//////////////////////////////////////////AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP///////////////////////////////////////////kBkqDud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdItjYW5fcmVzb2x2ZbkBIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEA//////////////////////////////////////////8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAbhAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALkBUGIAAGRiAACEkYCAgFF/uclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeoUYgABRFdQgFF/7nfmdqoCk7VwhfBIleas1RJ1aXkZ3la8nAyrmjkRcHQUYgAAr1dQYAEZUQBbYAAZWWAgAZCBUmAgkANgA4FSkFlgAFFZUmAAUmAA81tgAIBSYADzW1lZYCABkIFSYCCQA2AAGVlgIAGQgVJgIJADYAOBUoFSkFZbYCABUYBRkGAgAVFZUIGBklCSUFBgAGAAf2GS2cqxMuYrQBuOFT4dyEP+mfhZ7eN8SU3FcBw19rVkYAFZkIFSWWBgAZCBUmAgkAOEgVJgIJADhYFSYCCQA2DIgVJgAGAAWvGAUWAAFGIAASxXgFFgARRiAAE2V1BgARlRAFtQYABbkVBQkFZbYCABUWABkFBiAAEwVltQUIKRUFBiAACMVgq4YAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAguclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKBDRYn0GjHlYnm5qTVWMW+uP3ulOodmXRVscw6IZ0Iu33W3Ozo="
    }
  },
  "version": 1
}
```

After that a new state updated can be triggered.

#### Contract address computation
The created contract is part of the state tree. It has its own balance and its
place in the contracts subtree of the channel's state tree. In order to
inspect its balance or call the contract one needs its address.

Computation of this address is done exactly as it is in on-chain contracts -
it is a hashed version of the channel's owner pubkey and the nonce. Only
difference is that nonce is not computated in channels and the update round is
used instead.

### Call a contract
The call contract update is calling a preexisting contract inside the channel's internal state tree. The update is a change to be applied on top of the latest state. It has the
following structure.

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | contract | string | address of the contract to call |
  | abi_version | integer | version of the ABI |
  | amount | integer | amount the caller of the contract commits to it |
  | call_data | string | ABI encoded compiled AEVM call data for the code |

That would call a contract with the poster being the `caller` of it. Poster
commits an `amount` amount of tokens to the contract.

The call would also create a `call` object inside the channel state tree. It contains the result of the contract call.

#### Start call a contract update
##### Trigger a contract call update
The caller sends a message containing the desired change
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update.call_contract",
  "params": {
    "abi_version": 1,
    "amount": 0,
    "call_data": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACDud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATOUQxejJESkNMcUF3TUEudGVzdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABm9yYWNsZQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGjWjLw==",
    "contract": "ct_2BwwHijMMhHYSYfxHgwqKpWwcFfFGroTvoVMHoFXpEh9a6Mnht"
  }
}
```

##### Caller signs updated state
The caller receives a message containing the updated channel state as an
off-chain transaction

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+QHCOQGhBsYNEx0Z+hJ2sqygIgO1x7QvI4bfS5Tu+TpqeLgHdvDVefkBebkBdvkBc4ICPgGhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmoQWcjZC7BRy2SQGOiWHDx0RN+yafzkFu0uTGicKcd+FgawEAgw9CQAG5ASAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIO535naqApO1cIXwSJXmrNUSdWl5Gd5WvJwMq5o5EXB0AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABM5RDF6MkRKQ0xxQXdNQS50ZXN0AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGb3JhY2xlAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAoPA6RIoZgMpmOYLiaguYmMoyO7DAfH7uBqaQv+YEp/bxJ10a8w=="
    }
  },
  "version": 1
}
```

The caller is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "tx": "tx_+QIOCwH4QrhAn+ahmtP+kzvTxtoqpwpmDzfFmQL1EB/6fmx0jZhbeIMhkT8l6r7nk0aJz33HI7HXiHPICZbT+OqoQurzc7d9CbkBxfkBwjkBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1Xn5AXm5AXb5AXOCAj4BoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZqEFnI2QuwUctkkBjolhw8dETfsmn85BbtLkxonCnHfhYGsBAIMPQkABuQEgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACDud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATOUQxejJESkNMcUF3TUEudGVzdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABm9yYWNsZQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwKDwOkSKGYDKZjmC4moLmJjKMjuwwHx+7gamkL/mBKf28Ux3hJY="
  }
}
```

#### Acknowledger update
The acknowledger receives an info message indicating an upcoming change:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "update"
    }
  },
  "version": 1
}
```

Then the acknowledger receives a new message containing the updated channel state as an off-chain transaction
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update_ack",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+QHCOQGhBsYNEx0Z+hJ2sqygIgO1x7QvI4bfS5Tu+TpqeLgHdvDVefkBebkBdvkBc4ICPgGhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmoQWcjZC7BRy2SQGOiWHDx0RN+yafzkFu0uTGicKcd+FgawEAgw9CQAG5ASAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIO535naqApO1cIXwSJXmrNUSdWl5Gd5WvJwMq5o5EXB0AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABM5RDF6MkRKQ0xxQXdNQS50ZXN0AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGb3JhY2xlAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAoPA6RIoZgMpmOYLiaguYmMoyO7DAfH7uBqaQv+YEp/bxJ10a8w=="
    }
  },
  "version": 1
}
```

Note that this is the same transaction as the one that the caller had already signed. The acknowledger is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update_ack",
  "params": {
    "tx": "tx_+QIOCwH4QrhAAWS9EY0b29fSi0kmp/YSKWLfUYkQspiRngD7SKIWs1EIHZJlSYlbcsHO/1hgYDkLpeTb16Jd7ljFdpfGOyuZDLkBxfkBwjkBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1Xn5AXm5AXb5AXOCAj4BoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZqEFnI2QuwUctkkBjolhw8dETfsmn85BbtLkxonCnHfhYGsBAIMPQkABuQEgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACDud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATOUQxejJESkNMcUF3TUEudGVzdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABm9yYWNsZQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwKDwOkSKGYDKZjmC4moLmJjKMjuwwHx+7gamkL/mBKf28fQLRz4="
  }
}
```

#### Finish update
After both the parties have signed the new updated state of the channel - it is
considered the latest one. Corresponding update messages are sent to both
parties to indicate it. The payload of the message contains the latest
co-signed off-chain update so the participants can persist it locally.

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "state": "tx_+QJQCwH4hLhAAWS9EY0b29fSi0kmp/YSKWLfUYkQspiRngD7SKIWs1EIHZJlSYlbcsHO/1hgYDkLpeTb16Jd7ljFdpfGOyuZDLhAn+ahmtP+kzvTxtoqpwpmDzfFmQL1EB/6fmx0jZhbeIMhkT8l6r7nk0aJz33HI7HXiHPICZbT+OqoQurzc7d9CbkBxfkBwjkBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1Xn5AXm5AXb5AXOCAj4BoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZqEFnI2QuwUctkkBjolhw8dETfsmn85BbtLkxonCnHfhYGsBAIMPQkABuQEgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACDud+Z2qgKTtXCF8EiV5qzVEnVpeRneVrycDKuaORFwdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATOUQxejJESkNMcUF3TUEudGVzdAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABm9yYWNsZQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwKDwOkSKGYDKZjmC4moLmJjKMjuwwHx+7gamkL/mBKf28ZWCyeo="
    }
  },
  "version": 1
}
```

#### Getting a call result
All calls are stored in the channel state tree. In order to extract one out of
there and inspect it, one shall send a WebSocket event
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.get.contract_call",
  "params": {
    "caller": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
    "contract": "ct_2BwwHijMMhHYSYfxHgwqKpWwcFfFGroTvoVMHoFXpEh9a6Mnht",
    "round": 121
  }
}
```

The `contract_address` is the address of the contract that had been called, the `round` is the round of the update and
`caller_address` is the address of the caller.

Then the call is returned through an incoming message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.get.contract_call.reply",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "caller_id": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
      "caller_nonce": 121,
      "contract_id": "ct_2BwwHijMMhHYSYfxHgwqKpWwcFfFGroTvoVMHoFXpEh9a6Mnht",
      "gas_price": 1,
      "gas_used": 13126,
      "height": 121,
      "log": [],
      "return_type": "ok",
      "return_value": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAArMtts"
    }
  },
  "version": 1
}
```

It is worth mentioning that since this is an off-chain transaction the gas price specified is not consumed.
That amount of gas represents the amount of computations. It could be used for
aproximation for the gas needed for executing a contract on-chain if a similar amount of
computations are required. Computation heavy contracts might be just too
expensive to be force progressed on-chain, so please use with caution.

## <a name="leave-reestablish"></a>Optionally leave/reestablish
It is possible to leave a channel and then later reestablish the channel
off-chain state and continue operation. Leaving the channel can either be
done by simply disconnecting, or by sending a `'leave'` request. When receving
a leave request, the channel fsm passes it on to the peer fsm, reports the
current mutually signed state and then terminates. The `'reestablish'` request
is very similar to a [Channel open](#channel-open) request, but also requires
the channel id and the latest mutually signed state.

The full state, including state trees, is cached internally by the Aeternity
node, and upon reestablish, it is verified that the encoded state provided
by the client corresponds to the latest full state retrieved from the cache.

### Leave request
Example:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.leave",
  "params": {}
}
```

The fsm responds with the following type of report:
o```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.leave",
  "params": {
    "channel_id": "ch_2eZhEJ676pykdcHHcTZuSF346bq1ru3qnAiT2KowopwrPKwy7t",
    "data": {
      "state": "tx_+QENCwH4hLhABM97OxIYPtz2Oc4Kf20AMKA3GgnsQMtopM2g8ap17B1mZ9uFHB9LlM+rQPddSnhYIUj40JQq+tEJiVs7JEUdC7hA9Um2nQj7YlMx3ZC5UQgfF9Dg/FlPTQ5SkzSdAzEL0/dw9QuXKQXpPH6D0T2CZCxiqGvJqJykAqWofFWY8cnaCLiD+IEyAaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGP6olImAAoQFnHFVGRklFdbK0lPZRaCFxBmPYSJPN0tI2A3pUwz7uhIYkYTnKgAACCgCGEjCc5UAAwKCjPk7CXWjSHTO8V2Y9WTad6D/5sB8yCR8WumWh0WxWvwUG+R1a"
    }
  },
  "version": 1
}
```

### Reestablish
Open the channel in the same way as in the
[Initiator WebSocket open](#initiator-websocket-open) example,
adding the parameters `existing_channel_id` and `offchain_tx` with values
matching the ones provided in the `leave` report above. Some parameters (related to open transaction) are not required and ignored. See [Channel parameters](#channel-parameters):

```
$ wscat --connect
'localhost:3014/channel?protocol=json-rpc&initiator=ak_...&role=initiator&existing_channel_id=ch_FhYNM5KorNAcRwexe1CE3jH5DZd7FBD2g9XhBDHGEouDqVRCR&offchain_tx=tx_+QENCwH4hLhABM97OxIYPtz2Oc4Kf20AMKA3GgnsQMtopM2g8ap17B1mZ9uFHB9LlM+rQPddSnhYIUj40JQq+tEJiVs7JEUdC7hA9Um2nQj7YlMx3ZC5UQgfF9Dg/FlPTQ5SkzSdAzEL0/dw9QuXKQXpPH6D0T2CZCxiqGvJqJykAqWofFWY8cnaCLiD+IEyAaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGP6olImAAoQFnHFVGRklFdbK0lPZRaCFxBmPYSJPN0tI2A3pUwz7uhIYkYTnKgAACCgCGEjCc5UAAwKCjPk7CXWjSHTO8V2Y9WTad6D/5sB8yCR8WumWh0WxWvwUG+R1a
```

The channel fsm responds with the following event reports if all goes well:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": null,
    "data": {
      "event": "channel_reestablished"
    }
  },
  "version": 1
}
```

then the standard report indicating that the channel is open:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2eZhEJ676pykdcHHcTZuSF346bq1ru3qnAiT2KowopwrPKwy7t",
    "data": {
      "event": "open"
    }
  },
  "version": 1
}
```

followed by an update report with the latest mutually-signed state:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "channel_id": "ch_2eZhEJ676pykdcHHcTZuSF346bq1ru3qnAiT2KowopwrPKwy7t",
    "data": {
      "state": "tx_+QENCwH4hLhABM97OxIYPtz2Oc4Kf20AMKA3GgnsQMtopM2g8ap17B1mZ9uFHB9LlM+rQPddSnhYIUj40JQq+tEJiVs7JEUdC7hA9Um2nQj7YlMx3ZC5UQgfF9Dg/FlPTQ5SkzSdAzEL0/dw9QuXKQXpPH6D0T2CZCxiqGvJqJykAqWofFWY8cnaCLiD+IEyAaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGP6olImAAoQFnHFVGRklFdbK0lPZRaCFxBmPYSJPN0tI2A3pUwz7uhIYkYTnKgAACCgCGEjCc5UAAwKCjPk7CXWjSHTO8V2Y9WTad6D/5sB8yCR8WumWh0WxWvwUG+R1a"
    }
  },
  "version": 1
}
```

## Channel mutual close
At any moment after the channel is opened, a closing procedure can be
triggered. This can be done by either of the parties. The process is similar to
the [off-chain updates](#channel-off-chain-update). The most notable change is the
special transaction co-signed by both parties. It is called
`channel_close_mutual_tx`. After gathering singatures it will end up on the
chain and has the following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | channel id | string | ID of the channel|
  | from | string | initiator's public key |
  | initiator_amount_final | integer | final amount of tokens to be awarded by the initiator |
  | responder_amount_final | integer | final amount of tokens to be awarded by the responder |
  | ttl | integer | maximum block height to include the transaction |
  | fee | integer | fee to be paid to the miner |
  | nonce | integer | initiator's nonce |

Since any of the participants can initiate a closing, we will use `starter`
for the peer that triggers the process and `acknowledger` for the other one.

### Initiate mutual close
The starter sends the following message and triggers the closing procedure:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.shutdown",
  "params": {}
}
```

### Starter signing
Then the starter receives a `channel_close_mutual_tx` to sign:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.shutdown_sign",
  "params": {
    "channel_id": "ch_UpJnk4rp1qMqze3PxDKNAPUtKR18MNkCoAx3wX8gkQy8JfsjM",
    "data": {
      "tx": "tx_+F01AaEGPyim63JiB1RogAr8VnTjD2/LeL6fGLhySWhqgr7e7XihAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmhjaR1q+//4YbSOtX4AEAhhIwnOVAAANe4ar8"
    }
  },
  "version": 1
}
```

Starter is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.shutdown_sign",
  "params": {
    "tx": "tx_+KcLAfhCuEBjFLnPgoPp15I2oYzgPjrQJQyqvg1O1EimmGosw9jrNw2jTfaZXzSleSqPJ7F5T/dqLUSmqN5Py7ZlWQFKn4IFuF/4XTUBoQY/KKbrcmIHVGiACvxWdOMPb8t4vp8YuHJJaGqCvt7teKEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGNpHWr7//hhtI61fgAQCGEjCc5UAAA9iOZm4="
  }
}
```

### Acknowledger signing
Then the acknowledger receives a `channel_close_mutual_tx` to sign:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.shutdown_sign_ack",
  "params": {
    "channel_id": "ch_UpJnk4rp1qMqze3PxDKNAPUtKR18MNkCoAx3wX8gkQy8JfsjM",
    "data": {
      "tx": "tx_+F01AaEGPyim63JiB1RogAr8VnTjD2/LeL6fGLhySWhqgr7e7XihAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmhjaR1q+//4YbSOtX4AEAhhIwnOVAAANe4ar8"
    }
  },
  "version": 1
}
```

Note that this is the same transaction. The acknowledger is to decode the transaction, inspect its contents, sign it, encode it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.shutdown_sign_ack",
  "params": {
    "tx": "tx_+KcLAfhCuECzm2E8CsyzKvbO6zx8QHMnqX2lorQfH/t+FE7SQsf7tAL32BbqyFMox0su5RN9nriuXW1zdSCBkNvtKGDyVukCuF/4XTUBoQY/KKbrcmIHVGiACvxWdOMPb8t4vp8YuHJJaGqCvt7teKEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGNpHWr7//hhtI61fgAQCGEjCc5UAAA3BBwE8="
  }
}
```

#### Signed channel_close_mutual_tx
Both participants receive the co-signed `channel_close_mutual_tx`:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.on_chain_tx",
  "params": {
    "channel_id": "ch_UpJnk4rp1qMqze3PxDKNAPUtKR18MNkCoAx3wX8gkQy8JfsjM",
    "data": {
      "tx": "tx_+OkLAfiEuEBjFLnPgoPp15I2oYzgPjrQJQyqvg1O1EimmGosw9jrNw2jTfaZXzSleSqPJ7F5T/dqLUSmqN5Py7ZlWQFKn4IFuECzm2E8CsyzKvbO6zx8QHMnqX2lorQfH/t+FE7SQsf7tAL32BbqyFMox0su5RN9nriuXW1zdSCBkNvtKGDyVukCuF/4XTUBoQY/KKbrcmIHVGiACvxWdOMPb8t4vp8YuHJJaGqCvt7teKEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGNpHWr7//hhtI61fgAQCGEjCc5UAAA3KhgIw="
    }
  },
  "version": 1
}
```

Using its hash, participants can track its progress on the chain: entering the mempool, block inclusion and a number of confirmations.

### Channel closing
After both parties have received the co-signed `channel_close_mutual_tx` transaction, it is posted on the chain and the microservice handling the off-chain
requests dies. Parties receive the following info:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_UpJnk4rp1qMqze3PxDKNAPUtKR18MNkCoAx3wX8gkQy8JfsjM",
    "data": {
      "event": "died"
    }
  },
  "version": 1
}
```

Then the WebSocket connection is closed.

### Tracking the progress of the onchain transaction
After calculating the hash of the co-signed `channel_close_mutual_tx` parties can track
its progress as they would do with any on-chain transaction
```
curl 'http://127.0.0.1:3013/v2/transactions/th_2qkN973cNJiejXVJoXkXbttf1iKetWJCSY1W5VUBh3pnRS1kCC'
```
if the `block_hash` is `none` - then the transaction is still in the mempool.

### Other WebSocket events
#### Update error
Updates are not always successful, for example one participant tries to spend
more tokens that one currently has in the channel's balance. This diverges
from the update flow [described above](#channel-off-chain-update).

Example message for when the `from` does not have enough tokens to spend
```javascript
{
  "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
  "error": {
    "code": 3,
    "data": [
      {
        "code": 1001,
        "message": "Insufficient balance"
      }
    ],
    "message": "Rejected",
    "request": {
      "jsonrpc": "2.0",
      "method": "channels.update.new",
      "params": {
        "amount": 10000000000000000,
        "from": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
        "to": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC"
      }
    }
  },
  "id": null,
  "jsonrpc": "2.0",
  "version": 1
}
```

The structure is having a `reason` and `request` holding the request being
sent. Possible error reasons are:

* `insufficient_balance` - when `from` does not have enough tokens in the
  channel. Keep in mind that there is a minimal amount of `channel_reserve`
  tokens to be kept by both parties.

* `negative_amount` - the `udpate` event contained a negative amount

* `invalid_pubkeys` - at least one of the addresses in the `update` event is
  not present in the channel.

#### Update conflict

Since updates can be triggered by either party, it is possible both
participants to start an `update` almost simultaneously. If a new `update` is
started by a participant while the other participant has started an `update` of
ones' own - a conflict occurs. Then both `update`-s are invalidated and the
state is reverted to the last mutually signed one. Both participant receive a
message containing a reference to the correct state.
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.conflict",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
      "round": 5
    }
  },
  "version": 1
}
```

#### Generic messages

There is the functionality to send participants messages containing
information. In the scope of this - we are going to be calling those a `sender`
and a `receiver`. These roles can be taken by any of the participants, anytime.

The `sender` pushes a message with the following structure:

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.message",
  "params": {
    "info": "hejsan",
    "to": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC"
  }
}
```

Then the `receiver` gets an event containing the info being sent and some
details:

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.message",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "message": {
        "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
        "from": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
        "info": "hejsan",
        "to": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC"
      }
    }
  },
  "version": 1
}
```

#### Total balance update events

After the channel has been opened it has a total balance of tokens committed to
it. This balance is persisted as part of the on-chain channel state. Upon
closing a channel on-chain, the closing balances of the participants are
checked against this balance. Under no circumstances can the sum of the closing balances
be greater than the total balance on-chain.

Participants are able to modify the total balance: the following two functionalities are available:

* deposit - when a participant wants to commit more tokens from his on-chain
  balance to the channel total balance
* withdrawal - when a participant wants to transfer tokens out of the channel
  on-chain balance to one's personal account

### Deposit events

After the channel had been opened any of the participants can initiate a
deposit. The process closely resembles the [update](#update). The most notable
difference is the transaction has been co-signed: it is `channel_deposit_tx` and
after the procedure is finished, it is posted on-chain.

Since both the initiator and responder can deposit tokens, in the scope of this description we
will call the participant that commits tokens to the channel a depositor and
the other party - acknowledger. Note that any public key outside of the channel participants
is considered invalid for the depositor role.

#### Deposit transaction
The `channel_deposit_tx` is a change to be applied on top of the latest channel state. It also is
posted on-chain and is included in a block. It has the following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | channel id | string | ID of the channel|
  | from | string | depositor's public key |
  | amount | integer | the amount committed to the channel |
  | ttl | integer | minimum block height to include the transaction |
  | fee | integer | fee to be paid to the miner |
  | state_hash | string | the root of the internal channel state hash after the deposit |
  | round | integer | the next channel round |
  | nonce | integer | depositor's nonce |

#### Start deposit

Any of the participants can initiate a deposit. Only requirements are:
* Channel is already opened
* No off-chain update/deposit/withdrawal is currently being performed
* Channel is not being closed or in a solo closing state
* The deposit must be equal to or greater than zero, and cannot exceed the
  available balance on the depositor's account

##### Trigger a deposit

The depositor sends a WebSocket message containing the desired change
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.deposit",
  "params": {
    "amount": 2
  }
}
```

##### Depositor signs updated state
The depositor receives a message containing the updated channel state as a
`channel_deposit_tx` transaction
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.deposit_tx",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+HIzAaEGxg0THRn6EnayrKAiA7XHtC8jht9LlO75Omp4uAd28NWhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmAgCGEjCc5UAAoIGnTOuUnVM9QThcVqCCAFeJ51D7a6A4ldyhd3nuKg7lAgdGp4fe"
    }
  },
  "version": 1
}
```
The depositor is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.deposit_tx",
  "params": {
    "tx": "tx_+LwLAfhCuEA8SP3yaWa1Esu9cpKAAdMio0dkAvSdorjnz4wrdSx5HKIupPoLNHygNM23uv9IMnOiVLDGFUZL26UDwnmYCKsFuHT4cjMBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACggadM65SdUz1BOFxWoIIAV4nnUPtroDiV3KF3ee4qDuUCB0cHknY="
  }
}
```

#### Acknowledger update
The acknowledger receives an info message indicating an upcoming change:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "deposit_created"
    }
  },
  "version": 1
}
```

Then the acknowledger receives a new message containing the deposit
transaction for confirmation:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.deposit_ack",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+HIzAaEGxg0THRn6EnayrKAiA7XHtC8jht9LlO75Omp4uAd28NWhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmAgCGEjCc5UAAoIGnTOuUnVM9QThcVqCCAFeJ51D7a6A4ldyhd3nuKg7lAgdGp4fe"
    }
  },
  "version": 1
}
```

Note that this is the same transaction as the one that the depositor had already signed. The acknowledger is to decode the transaction, inspect its contents, sign it, encode it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.deposit_ack",
  "params": {
    "tx": "tx_+LwLAfhCuEB2IeqEF8suaKRzZCtzm2Ux2luV/FxDyqXd2i6ePUowdJhT6V1oqUFTcbOo3eh+fmZh3HDVyLFkckVR/192AVsFuHT4cjMBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACggadM65SdUz1BOFxWoIIAV4nnUPtroDiV3KF3ee4qDuUCB4YWJuc="
  }
}
```

#### Finish deposit
After both parties had signed the deposit transaction, the transaction is posted on-chain and both parties receive it:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.on_chain_tx",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+P4LAfiEuEA8SP3yaWa1Esu9cpKAAdMio0dkAvSdorjnz4wrdSx5HKIupPoLNHygNM23uv9IMnOiVLDGFUZL26UDwnmYCKsFuEB2IeqEF8suaKRzZCtzm2Ux2luV/FxDyqXd2i6ePUowdJhT6V1oqUFTcbOo3eh+fmZh3HDVyLFkckVR/192AVsFuHT4cjMBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACggadM65SdUz1BOFxWoIIAV4nnUPtroDiV3KF3ee4qDuUCB50QwtI="
    }
  },
  "version": 1
}
```

They are both to compute the transaction hash. Using it,
participants can track its progress on the chain: entering the mempool, block
inclusion and a number of confirmations.

After the `minimum_depth` block confirmations each participant is informed
for the deposit progress on-chain by one's own node:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "own_deposit_locked"
    }
  },
  "version": 1
}
```

An update from one's own node that the other party had confirmed that the block
height needed is reached:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "deposit_locked"
    }
  },
  "version": 1
}
```

With this the deposit sequence is complete and the channel can continue with
other updates. Note that the deposit transaction's `round` and `state_hash`
are the ones considered latest from the channel's perspective. For example the
next correct off-chain update/deposit/withdrawal shall have a deposit
transaction's `round` plus one.

### Withdraw events

After the channel had been opened any of the participants can initiate a
withdrawal. The process closely resembles the [update](#update). The most notable
difference is that the transaction has been co-signed: it is `channel_withdraw_tx` and
after the procedure is finished - it is being posted on-chain.

Since both the initiator and responder can withdraw tokens, in the scope of this description we
will call the participant that commits tokens to the channel a withdrawer and
the other party - an acknowledger. Note that any public key outside of the channel participants
is considered invalid for the withdrawer role.

#### Withdraw transaction
The `channel_withdraw_tx` is a change to be applied on top of the latest channel state. It also is
posted on-chain and is included in a block. It has the following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | channel id | string | ID of the channel|
  | to | string | withdrawer's public key |
  | amount | integer | the amount taken out from the channel |
  | ttl | integer | minimum block height to include the transaction |
  | fee | integer | fee to be paid to the miner |
  | state_hash | string | the root of the internal channel state hash after the withdraw |
  | round | integer | the next channel round |
  | nonce | integer | withdrawer's nonce |

#### Start withdraw

Any of the participants can initiate a withdrawal. The only requirements are:
* Channel is already opened
* No off-chain update/deposit/withdrawal is currently being performed
* Channel is not being closed or in a solo closing state
* The withdrawal amount must be equal to or greater than zero, and cannot
  exceed the available balance on the channel (minus the `channel_reserve`)

##### Trigger a withdrawal

The withdrawer sends a WebSocket message containing the desired change
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.withdraw",
  "params": {
    "amount": 2
  }
}
```

##### Withdrawer signs updated state
The withdrawer receives a message containing the updated channel state as a
`channel_withdraw_tx` transaction
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.withdraw_tx",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+HI0AaEGxg0THRn6EnayrKAiA7XHtC8jht9LlO75Omp4uAd28NWhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmAgCGEjCc5UAAoEUfp6Gv3gLYNHIy08ITGVY9Xv7I69D5Yu0FUzyRePtuBAiB1DYi"
    }
  },
  "version": 1
}
```

The withdrawer is to decode the transaction, inspect its contents, sign it, encode
it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.withdraw_tx",
  "params": {
    "tx": "tx_+LwLAfhCuEDWK5IT8u7+ThM6U1uF13WsMsBBuQWvA35wXJeCH+PidpjeivQD4vB3imhRQo8gPPSbNkbvolMajwVYP8BpxzYEuHT4cjQBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACgRR+noa/eAtg0cjLTwhMZVj1e/sjr0Pli7QVTPJF4+24ECFbANeQ="
  }
}
```

#### Acknowledger update
The acknowledger receives an info message indicating an upcoming change:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "withdraw_created"
    }
  },
  "version": 1
}
```

Then the acknowledger receives a new message containing the withdraw
transaction for confirmation:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.withdraw_ack",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+HI0AaEGxg0THRn6EnayrKAiA7XHtC8jht9LlO75Omp4uAd28NWhAbG1d7zTJ8s55V5sAmvWp0obNd5sBlDErlHvq3WeQVtmAgCGEjCc5UAAoEUfp6Gv3gLYNHIy08ITGVY9Xv7I69D5Yu0FUzyRePtuBAiB1DYi"
    }
  },
  "version": 1
}
```

Note that this is the same transaction as the one that the withdrawer has already signed. The acknowledger is to decode the transaction, inspect its contents, sign it, encode it and then post it back via a WebSocket message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.withdraw_ack",
  "params": {
    "tx": "tx_+LwLAfhCuEDZd+OFg28UyfwTEDaw2JQ23oL8/Y8G06oi2YgSmG6pmvH0DbP/mbyTo7ZI5Jzv4veS4MrUQSor0d6Q9fWX8GoKuHT4cjQBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACgRR+noa/eAtg0cjLTwhMZVj1e/sjr0Pli7QVTPJF4+24ECPo7AIM="
  }
}
```

#### Finish withdrawal
After both the parties had signed the withdraw transaction, the transaction is posted on-chain and
both parties receive it:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.on_chain_tx",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+P4LAfiEuEDWK5IT8u7+ThM6U1uF13WsMsBBuQWvA35wXJeCH+PidpjeivQD4vB3imhRQo8gPPSbNkbvolMajwVYP8BpxzYEuEDZd+OFg28UyfwTEDaw2JQ23oL8/Y8G06oi2YgSmG6pmvH0DbP/mbyTo7ZI5Jzv4veS4MrUQSor0d6Q9fWX8GoKuHT4cjQBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACgRR+noa/eAtg0cjLTwhMZVj1e/sjr0Pli7QVTPJF4+24ECGAF7UQ="
    }
  },
  "version": 1
}
```

They are both to compute the transaction hash. Using it,
participants can track its progress on the chain: entering the mempool, block
inclusion and a number of confirmations.

After the `minimum_depth` block confirmations each participant is informed
for the withdraw progress on-chain by one's own node:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "own_withdraw_locked"
    }
  },
  "version": 1
}
```

An update from one's own node that the other party had confirmed that the block
height needed is reached:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.info",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "event": "withdraw_locked"
    }
  },
  "version": 1
}
```

With this the withdrawal sequence is complete and the channel can continue with
other updates. Note that the withdraw transaction's `round` and `state_hash`
are the ones considered latest from the channel's perspective. For example the
next correct off-chain update/deposit/withdrawal shall have a withdraw
transaction's `round` plus one.

### Getting state
At any moment in time any participant can ask one's own FSM for various views of
the latest channel state. This is to help wallet apps but they shall not trust
FSM and compute state locally.

#### Get balances
In order to get the balances as those are part from the channel's state tree, a
participant sends a WebSocket message
```javascript
{
  "id": -576460752303423474,
  "jsonrpc": "2.0",
  "method": "channels.get.balances",
  "params": {
    "accounts": [
      "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC",
      "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm"
    ]
  }
}
```

The `accounts` section of the payload contains a list of addresses to fetch
balances of. Those can be either account balances or a contract ones, encoded
as an account addresses.

A response of this call looks like
```javascript
{
  "channel_id": "ch_2TPA12qYnQUE6359Vmg55SD5wMvKUQxZA9wWChWSsRkFvvYAUd",
  "id": -576460752303423474,
  "jsonrpc": "2.0",
  "result": [
    {
      "account": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC",
      "balance": 40000000000000
    },
    {
      "account": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
      "balance": 70000000000000
    }
  ],
  "version": 1
}
```

If a certain account address had not being found in the state tree - it is simply
skipped in the response.

#### Get proof of inclusion
In order to build and use different services, one might need to provide a third party
a minimal view of the internal channel's state.

In order to fetch a proof of inclusion from the latest modified state tree,
a participant sends a WebSocket message
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.get.poi",
  "params": {
    "accounts": [
      "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
      "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC"
    ],
    "contracts": [
      "ct_2Yy7TpPUs7SCm9jkCz7vz3nkb18zs78vcuVQGbgjRaWQNTWpm5"
    ]
  }
}
```

The `accounts` section of the payload contains a list of addresses to include in the
proof of inclusion. Almost the same goes with the contract addesses listed in `contracts` section:
the only difference being that contract's accounts will be automatically be added
to proof of inclusion as well as their state.

A response of this call looks like
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.get.poi.reply",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "poi": "pi_+QjFPAH5Aar5AaegUHXVJYllXPYCKsNzFPm556q8IgudzPRI4O/BDFpobNP5AYP4lKBQddUliWVc9gIqw3MU+bnnqrwiC53M9Ejg78EMWmhs0/hxgICAgICAoO0CFjxibIfdehu8wgGaVHmkULhlWpsI0fCWsUChDyMJgICAgKCQ1bLD69k9BcQwVvuyoPXBJe++idIJ/lwoO4qPYlt0YaCmyjgxBha42v7WL6pOfqG1Ttf6NYVnqZHqR+T0JmGLxYCAgID4T6CQ1bLD69k9BcQwVvuyoPXBJe++idIJ/lwoO4qPYlt0Ye2gMbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aLygoBAIY/qiUiX/X4SaCmyjgxBha42v7WL6pOfqG1Ttf6NYVnqZHqR+T0JmGLxeegPEg2CnIUGzfwXVy9z4FuBgcGBoC5MzyBosNeYBcMoZaFxAoBAAr4T6DtAhY8YmyH3XobvMIBmlR5pFC4ZVqbCNHwlrFAoQ8jCe2gNxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oSLygoBAIYkYTnKgAHj4qAjoqXI5ZEb6//BQUWWVlPkHd4lWRhzVFmXY/6F2X4ZBMDA+Qbs+QbpoBM+jdiLV6qhvwLfo/bK9PwOB9Zk8f4j3I25MmoWLu2g+QbF+IagCHQOBl3W9IhcdkmZJCJrKJ+h0G8+ZeIKUZ3W5p9/Nez4YyC4YAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP///////////////////////////////////////////hmoBM+jdiLV6qhvwLfo/bK9PwOB9Zk8f4j3I25MmoWLu2g+EOhAMxINgpyFBs38F1cvc+BbgYHBgaAuTM8gaLDXmAXDKGWoKwRE5jjO51HU4syeM79QdOg9AzlL1uevldJ6GXQlgV9+ESgJ6ky/mLaY+D64bXs7TR5BIRxO0hOakcKlSHrfu7sP7/iEKA1jHy+JkYGJXeg4QZT9QYTyIjuaukMdPKoxXeDh8vbz/hToDWMfL4mRgYld6DhBlP1BhPIiO5q6Qx08qjFd4OHy9vP8aCFlE6Nir08G/FQ+dPMIRjPZKt+jIKfH4aEhiH2DCC/C4CAgICAgICAgICAgICAgAD4RKA/cNKMPbnMACszBp/R0S52OAbwRhxnBpoCOSMcfJ92pOIgoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA+HSghZROjYq9PBvxUPnTzCEYz2SrfoyCnx+GhIYh9gwgvwv4UaA/cNKMPbnMACszBp/R0S52OAbwRhxnBpoCOSMcfJ92pKAIdA4GXdb0iFx2SZkkImson6HQbz5l4gpRndbmn3817ICAgICAgICAgICAgICAgPkEe6CsEROY4zudR1OLMnjO/UHToPQM5S9bnr5XSehl0JYFffkEV4CgJ6ky/mLaY+D64bXs7TR5BIRxO0hOakcKlSHrfu7sP7+AgICAgICAgICAgICAgLkEJPkEISgBoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZoMDAAG5A/L5A+9GAaD+6SgUyLZEFQgM0dnekwzMdKs+z+4qTA7+8R/txGeK2vkC+/kBKqBo8mdjOP9QiDmrpHdJ7/qL6H7yhPIH+z2ZmHAc1TiHxYRtYWluuMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoP//////////////////////////////////////////AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC4QAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD5AcuguclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeqEaW5pdLhgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA///////////////////////////////////////////uQFAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAFAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQD//////////////////////////////////////////wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD//////////////////////////////////////////+4zGIAAGRiAACEkYCAgFF/uclW8osxSan1mHqlBfPaGyIJzFc5I0AGK7bBvZ+fmeoUYgAAwFdQgFF/aPJnYzj/UIg5q6R3Se/6i+h+8oTyB/s9mZhwHNU4h8UUYgAAr1dQYAEZUQBbYAAZWWAgAZCBUmAgkANgA4FSkFlgAFFZUmAAUmAA81tgAIBSYADzW1lZYCABkIFSYCCQA2AAGVlgIAGQgVJgIJADYAOBUoFSkFZbYCABUVFZUICRUFCAkFCQVltQUIKRUFBiAACMVoABwArAwBYGkNE="
    }
  },
  "version": 1
}
```

If a certain address of an account or a contract is not found in the state tree -
the response is an error.

#### Dry-run a contract
In order to get the result of a potential contract call, one might need to
dry-run a contract call. It takes the exact same arguments as a call would and
returns the `call` object.

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.dry_run.call_contract",
  "params": {
    "abi_version": 1,
    "amount": 0,
    "call_data": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACBJ7EkHbAIDcSakMwPq3OQ7LiwylFexE8UKfiLciYCUtwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYPjVAQ==",
    "contract": "ct_2A67iNjuNd2erJdzDMCzVeJkj82cS1krGGFbQeheBhFELktpo4"
  }
}
```

The `payload` is an mirror image of the [call contract
update](#trigger-a-contract-call-update), the only difference being that the
dry-run does not impact the state and it does not need signing. The call is
executed in the channel's state but it does not impact the state whatsoever.
It uses as an environment the latest channel's state and the current top of
the blockchain as seen by the node.

A response to this call looks like
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.dry_run.call_contract.reply",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "caller_id": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
      "caller_nonce": 11,
      "contract_id": "ct_2A67iNjuNd2erJdzDMCzVeJkj82cS1krGGFbQeheBhFELktpo4",
      "gas_price": 1,
      "gas_used": 220,
      "height": 11,
      "log": [],
      "return_type": "ok",
      "return_value": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABX4y1tk"
    }
  },
  "version": 1
}
```

#### Get contract calls
Each participant persists contract call results locally. It is not required of
both participants to share the same list of contract calls as this does not
impact consensus between them. Any participant can prune his local set of
calls in order to free some memory. In order to inspect the result value of a
contract call, a participant sends a WebSocket message
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.get.contract_call",
  "params": {
    "caller": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
    "contract": "ct_2A67iNjuNd2erJdzDMCzVeJkj82cS1krGGFbQeheBhFELktpo4",
    "round": 10
  }
}
```

The combination of a `caller`, `contract` and a `round` of execution determines the
contract call. Providing an incorrect set of those results in an error response.

A non-error response of this call looks like this
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.get.contract_call.reply",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "caller_id": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
      "caller_nonce": 10,
      "contract_id": "ct_2A67iNjuNd2erJdzDMCzVeJkj82cS1krGGFbQeheBhFELktpo4",
      "gas_price": 1,
      "gas_used": 220,
      "height": 10,
      "log": [],
      "return_type": "ok",
      "return_value": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABX4y1tk"
    }
  },
  "version": 1
}
```

It is worth mentioning that the gas is not consumed, because this is an off-chain contract call.
It would be consumed if it were a on-chain one. This could happen if a call with a
similar computation amount is to be forced on-chain.

#### Pruning contract calls

Contract calls are kept locally in order for the participant to be able to
look them up. They consume memory and in order for the participant to free it -
one can prune all messages. This cleans up all locally stored contract calls
and those will no longer be available for fetching and inspection.

In order to prune local calls, a participant sends the following WebSocket
message:

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.clean_contract_calls",
  "params": {}
}
```

Once calls are pruned, the same participant receives the following message:
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.calls_pruned.reply",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "action": "calls_pruned"
    }
  },
  "version": 1
}
```

### Solo closing sequence
At any moment in time after the channel had been opened any participant can
initiate a solo closing. The mutual closing takes just one block
inclusion for the effects to take place. The solo closing sequence requires a
couple of blocks and at least 2 transaction fees to be paid. This makes the
solo closing sequence both slower and more expensive. It is intended to be
used when the other party is trying to cheat or is not responding for a while.
This is called a dispute and it is taken to the chain to resolve it. Dispute
resolution has the following steps:

1. single `channel_solo_close` transaction
2. zero or a couple of `channel_slash` transactions
3. single `channel_settle` transaction

The second step is not required and a `channel_solo_close` could be followed
either by zero, one or more `channel_slash` transactions, each subsequent one
presenting a newer state and overwriting the previous one. Those are settled by
a `channel_settle` transaction that finally closes the channel. Let's discuss
those in detail.

#### Payload and proof of inclusion
The idea behind `channel_solo_close` and `channel_settle` is for parties to
provide, on-chain, the latest channel internal state so that the channel can be closed.
First comes the `channel_solo_close` that provides some off-chain state. Then a
`channel_slash` can be posted but it is checked that it has a newer state than the
`channel_solo_close` one. Then parties can post more `channel_slash` transactions
but those are always checked to be containing a newer channel state than the last
received on-chain. If one party tries to cheat by posting some old state - the other
party can present to the chain a newer channel state and this overwrites the previous
posted one. Thus the comparison on channel states is important. This is done
by comparing rounds.

Both `channel_solo_close` and `channel_slash` contain a `payload` field. This
is either a binary containing a `channel_offchain` transaction or an empty
binary.

If it is a `channel_offchain` transaction - it must be co-signed by both
participants. It also contains a `channel_id`, `round` and `state_hash`.
The `channel_id` in combination with the correct singatures verifies that
this off-chain transaction indeed is part of the channel off-chain state. The
`round` represents the height of the channel's state at the time the
transaction was co-signed by the parties. The higher the round, the newer the
transaction is. This `round` must be greater than the last on-chain one for that channel.
The `state_hash` is the internal channel state tree root hash
at that `round` height.

If the transaction's `payload` is empty - then the latest on-chain state for
this channel is used. Both `channel_deposit` and `channel_withdraw`
transactions contain a `round` and a `state_hash` and the latest received one
overwrites the previous one. If there had been none of those, then the
`channel_create` transaction is used: it has a `state_hash` and an implicit
`round = 1`.

Either by having a value in the `payload` or not having one, the
`channel_solo_close` and `channel_slash` provide a channel's `round` and a `state_hash`.
In order to determine the order of the channel's states received - we compare
the `rounds` and keep the state with the greatest `round`, considered to be
the _newest_ and _latest_ state. They also provide the `state_hash` the
channel's state tree had at this `round`.

Both `channel_solo_close` and `channel_slash` contain a `poi` field. This is
the proof of inclusion for participants' balances in the channel state: all
the insignificant data in the channel's MPT (Matricia Perkel Tree) is replaced
by corresponding hashes.
The root hash of the PoI must be equal to the `state_hash` provided by the `payload`.
This guarantees that the PoI indeed is a proof of inclusion for tree at this height.

#### Solo close on-chain transaction
The `channel_close_solo` transaction is the one that triggers the solo closing
sequence. After it is included on-chain channel enters a _closing_ state and
any subsequent withdrawal or deposits are considered invalid. Preconditions for
the `channel_close_solo` to be valid are:

* channel is opened on-chain
* channel is not in a _closing_ state but not yet closed - no `channel_close_solo` has been
  included in a block yet

Any participant in the channel can post a `channel_close_solo` transaction. In
the scope of this description we will call the one that posts the transaction
_solo closer_.
The transaction has the following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | channel id | string | ID of the channel to close |
  | from | string | solo closer's public key |
  | payload | binary | closing payload |
  | poi | binary | closing proof of inclusion |
  | ttl | integer | maximum block height to include the transaction |
  | fee | integer | fee to be paid to the miner |
  | nonce | integer | solo closer's nonce |

`payload` and `poi` are validated as [described above](#payload-and-proof-of-inclusion)

#### Slash on-chain transaction
After the channel is already in a _closing_ state, both participants can provide
a newer state via the `channel_slash` transaction. Preconditions for
the `channel_slash` to be valid are:

* channel is opened on-chain
* channel is still in a _closing_ state - no `channel_settle` has been
  included in a block yet

Any participant in the channel can post a `channel_slash` transaction. In
the scope of this description we will call the one that posts the transaction
_slasher_.
The transaction has the following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | channel id | string | ID of the channel to slash |
  | from | string | slasher's public key |
  | payload | binary | slashing payload |
  | poi | binary | slashing proof of inclusion |
  | ttl | integer | maximum block height to include the transaction |
  | fee | integer | fee to be paid to the miner |
  | nonce | integer | slasher's nonce |

`payload` and `poi` are validated as [described above](#payload-and-proof-of-inclusion)

#### Settle on-chain transaction
After the `channel_close_solo` and all the `channel_slash` transactions,
it is time to finally close the channel. One of the participants posts a
`channel_settle` transaction that enforces closing of the channel. This
happens according to the latest channel state that was sent on-chain. The
`channel_settle` just finalizes the channel closing with the last received
state, redistributes tokens to participants and closes the channel. No further
disputes are possible after that.


In order to give parties time to slash a closing channel and update its state
with a newer one, there is a timeframe only after which the `channel_settle`
can be posted. This is measured in blocks mined on top of the last received
transaction for that channel (either `channel_close_solo` or `channel_slash`).
The amount itself is prenegotiated before opening the channel and is part of
the `channel_create` transaction - it is the value of `lock_period`. Under no
condition a `channel_settle` can be included in a block before passing the
`lock_period` amount of blocks on top of the last `channel_close_solo` or
`channel_slash` transaction. Every next included `channel_slash` restarts the
timer. It is worth noting that since those transactions must include a `payload`
newer than the prevous on-chain one - this timer can not be postponed
indefinitely. Preconditions for the `channel_settle` to be valid are:

* channel is opened on-chain
* channel is still in a _closing_ state - no other `channel_settle` has been
  included in a block yet
* at least `lock_period` blocks has been mined on top of the last
  `channel_close_solo` or `channel_slash`

Any participant in the channel can post a `channel_settle` transaction. In
the scope of this description we will call the one that posts the transaction
_settler_.
The transaction has the following structure:

  | Name | Type | Description |
  | ---- | ---- | ----------- |
  | channel id | string | ID of the channel to settle |
  | from | string | settler's public key |
  | initiator_amount_final | integer | initiator final amount |
  | responder_amount_final | integer | responder final amount |
  | ttl | integer | maximum block height to include the transaction |
  | fee | integer | fee to be paid to the miner |
  | nonce | integer | settler's nonce |

The amounts are the exact amounts stored in the channel object on-chain.
