[back](./README.md)
# State channel WebSocket API

Messages on the WebSocket API have to follow JSON-RPC specification version 2.0. See: [www.jsonrpc.org](https://www.jsonrpc.org/specification)

The WebSocket API provides the following actions:
 * [Off-chain update](#update)
 * [On-chain deposit](#deposit)
 * [On-chain withdrawal](#withdrawal)
 * [Contracts](#contracts)
 * [Generic message](#generic-message)
 * [Close mutual](#close-mutual)
 * [Leave](#leave)
 * [On-chain transactions](#on-chain-transactions)
 * [Info messages](#info-messages)

## Update
Roles:
 * Sender
 * Acknowledger

### Sender trigger an update
 * **method:** `channels.update.new`
 * **params:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | from | string | Participant's account to take tokens from | Yes |
  | to | string | Participant's account to add tokens to | Yes |
  | amount | integer | Amount of tokens to transfer | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update.new",
  "params": {
    "amount": 2,
    "from": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm",
    "to": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC"
  }
}
```

### Sender receives unsigned off-chain state
 * **method:** `channels.sign.update`
 * **params:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_offchain` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_+JU5AaEGxg0THRn6EnayrKAiA7XHtC8jht9LlO75Omp4uAd28NUG+E24S/hJggI6AaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2ahAWccVUZGSUV1srSU9lFoIXEGY9hIk83S0jYDelTDPu6EAaDVKneBkzhWnEbC2q9FakzUh62kF75skKy/zAal9PdC4a/QsrE="
    }
  },
  "version": 1
}
```

### Sender signed off-chain state responsse
 * **method:** `channels.update`
 * **params:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_offchain` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update",
  "params": {
    "tx": "tx_+N8LAfhCuECpOlAuBnwgezZlvmIZkD2Yw4xheb/BTnZaz+jAv2brWWoaUl95yOwZ6ataFXQmXvf3gVB7nGmWljxpHw0wwfIAuJf4lTkBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1Qb4TbhL+EmCAjoBoQGxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZqEBZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oQBoNUqd4GTOFacRsLar0VqTNSHraQXvmyQrL/MBqX090LhsL9soA=="
  }
}
```

### Acknowledger receives unsigned off-chain state
 * **method:** `channels.sign.update_ack`
 * **params:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_offchain` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.sign.update_ack",
  "params": {
    "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
    "data": {
      "tx": "tx_21uV5so71tzLyBzTGBd5qd318n8Z33ninWoJzuBBKa3y3cLv8jL1gBsUpwoT1Wzs57fpxgxk7asMpuxcKpYxRnH1Qk679DjPUjLx3Lu6eNnPnfDwb4NpMq5tmm1Sq1j4MLfi6mFLadQ4CyuiENcytACQgkiU2CP8jWHKDCxAprKxP7EnXRKGbyaXkQRjvxmd5BK5XpnPHMoLb4zrrQfex5Wi8SkJjxWrhRyTr7u8jqyyebVPYmz3iRnnoEHfiECzBLdAYBz12U4VgUNrYug8C3ns5GcB1ytaUmggpDGY4K97dyYR8aMorFfqY6rPjwpRoL1BjbJgUBw54VVgMEijfeVCNcyw8wrVJnZeUAQKSesJcPhWShY717GVeQfGGHLzJhTY7iYBUUQCLfoVms86jJ3vMo1d9DpnahpCXfrZeR2PExg8Cn9DXc"
    }
  },
  "version": 1
}
```

### Acknowledger signed off-chain state responsse
 * **method:** `channels.update_ack`
 * **params:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_offchain` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update_ack",
  "params": {
    "tx": "tx_xCHADUvUikbjGRcBmj4YQkTJGCoGV6JQVdJW2FU1ZAYGhdayeCZerGqPWbRz4Eduq1KtjUbBJgdxSF3UKyChKMXne3dEDnChRdiUop4HYkHJ8GF3xQpbSspvST5qPTJqvcCstQCDXmJMLiYiWQ2hoPXL3a1qiiVmSwx2ztVuVqsEf1NsQCbMiNeJj8Uvrcp2FKN8TG2VoMTBTiMcCdLGXhX31EaLYTTDVyFTXGgFRUTdAsHgBjcQzm9hgQS75QjhKY7VtyUBCisUEQp8Dcr76rpdT1Qy9n8JYKboPkFZpY9DVx9We2hstbP3fjgZVLgDRAvLoC5YppVE7GZgUbRp6PMmbPUyc3qFYaxA82g7TzndipqnrKuuGzDjoPaM2w5evx1TvXAF5u1beac2kW7kJyKjLfLhjKQ8bnwBwcZ3WpdRfCVe55LtPwYEoZQJtdzojjVcuLmgJjbb2GDHioi8KXTasHre5oZKwkyYByMqzDafVTMT3kJqvdQG6HKAm8XGP6LGRsZFcpkn5jGtGbq7PRpTbAn1RbHWvRAEH"
  }
}
```


### Update conflict
 * **method:** `channels.conflict`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | channel_id | string | channel id | Yes |
  | round | integer | last correct state round | Yes |

#### Example
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

### Update error
 * **method:** `channels.error`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | reason | string | description of the error | Yes |
  | request | json | the failed request | Yes |

#### Example
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
        "from": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC",
        "to": "ak_2MGLPW2CHTDXJhqFJezqSwYSNwbZokSKkG7wSbGtVmeyjGfHtm"
      }
    }
  },
  "id": null,
  "jsonrpc": "2.0",
  "version": 1
}
```

## Deposit
Roles:
 * Depositor
 * Acknowledger

### Depositor trigger a update
 * **method:** `channels.deposit`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | amount | integer | Amount of tokens to deposit in the channel | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.deposit",
  "params": {
    "amount": 2
  }
}
```

### Depositor receives unsigned deposit transaction
 * **method:** `channels.sign.deposit_tx`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_deposit` transaction | Yes |

#### Example
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

### Depositor signed deposit response
 * **method:** `channels.deposit_tx`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_deposit` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.deposit_tx",
  "params": {
    "tx": "tx_+LwLAfhCuEA8SP3yaWa1Esu9cpKAAdMio0dkAvSdorjnz4wrdSx5HKIupPoLNHygNM23uv9IMnOiVLDGFUZL26UDwnmYCKsFuHT4cjMBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACggadM65SdUz1BOFxWoIIAV4nnUPtroDiV3KF3ee4qDuUCB0cHknY="
  }
}
```

### Acknowledger receives unsigned deposit transaction
 * **method:** `channels.sign.deposit_ack`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_deposit` transaction | Yes |

#### Example
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

### Acknowledger signed deposit responsse
 * **method:** `channels.deposit_ack`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_deposit` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.deposit_ack",
  "params": {
    "tx": "tx_+LwLAfhCuEB2IeqEF8suaKRzZCtzm2Ux2luV/FxDyqXd2i6ePUowdJhT6V1oqUFTcbOo3eh+fmZh3HDVyLFkckVR/192AVsFuHT4cjMBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACggadM65SdUz1BOFxWoIIAV4nnUPtroDiV3KF3ee4qDuUCB4YWJuc="
  }
}
```

## Withdrawal
Roles:
 * Withdrawer
 * Acknowledger

### Withdrawer trigger a update
 * **method:** `channels.withdraw`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | amount | integer | Amount of tokens to withdraw form the channel | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.withdraw",
  "params": {
    "amount": 2
  }
}
```

### Withdrawer receives unsigned withdraw transaction
 * **method:** `channels.sign.withdraw_tx`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_withdraw` transaction | Yes |

#### Example
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

### Withdrawer signed withdraw response
 * **method:** `channels.withdraw_tx`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_withdraw` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.withdraw_tx",
  "params": {
    "tx": "tx_+LwLAfhCuEDWK5IT8u7+ThM6U1uF13WsMsBBuQWvA35wXJeCH+PidpjeivQD4vB3imhRQo8gPPSbNkbvolMajwVYP8BpxzYEuHT4cjQBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACgRR+noa/eAtg0cjLTwhMZVj1e/sjr0Pli7QVTPJF4+24ECFbANeQ="
  }
}
```

### Acknowledger receives unsigned withdraw transaction
 * **method:** `channels.sign.withdraw_ack`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_withdraw` transaction | Yes |

#### Example
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

### Acknowledger signed withdraw responsse
 * **method:** `channels.withdraw_ack`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_withdraw` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.withdraw_ack",
  "params": {
    "tx": "tx_+LwLAfhCuEDZd+OFg28UyfwTEDaw2JQ23oL8/Y8G06oi2YgSmG6pmvH0DbP/mbyTo7ZI5Jzv4veS4MrUQSor0d6Q9fWX8GoKuHT4cjQBoQbGDRMdGfoSdrKsoCIDtce0LyOG30uU7vk6ani4B3bw1aEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2YCAIYSMJzlQACgRR+noa/eAtg0cjLTwhMZVj1e/sjr0Pli7QVTPJF4+24ECPo7AIM="
  }
}
```

## Contracts

### Contract create

 * **method:** `channels.update.new_contract`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | code | contract code | contract code | Yes |
  | call\_data | call data | call data for contract creation | Yes |
  | vm\_version | integer | contract virtual machine version (vm for which code was compiled) | Yes |
  | abi\_version | integer | contract virtual machine abi version | Yes |
  | deposit | integer | contract creation deposit | Yes |

 * **no response**

#### Example

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update.new_contract",
  "params": {
    "abi_version": 1,
    "call_data": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACC5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAnHQYrA==",
    "code": "cb_+QPvRgGg/ukoFMi2RBUIDNHZ3pMMzHSrPs/uKkwO/vEf7cRnitr5Avv5ASqgaPJnYzj/UIg5q6R3Se/6i+h+8oTyB/s9mZhwHNU4h8WEbWFpbrjAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKD//////////////////////////////////////////wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAuEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA+QHLoLnJVvKLMUmp9Zh6pQXz2hsiCcxXOSNABiu2wb2fn5nqhGluaXS4YAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP//////////////////////////////////////////7kBQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEA//////////////////////////////////////////8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA///////////////////////////////////////////uMxiAABkYgAAhJGAgIBRf7nJVvKLMUmp9Zh6pQXz2hsiCcxXOSNABiu2wb2fn5nqFGIAAMBXUIBRf2jyZ2M4/1CIOaukd0nv+ovofvKE8gf7PZmYcBzVOIfFFGIAAK9XUGABGVEAW2AAGVlgIAGQgVJgIJADYAOBUpBZYABRWVJgAFJgAPNbYACAUmAA81tZWWAgAZCBUmAgkANgABlZYCABkIFSYCCQA2ADgVKBUpBWW2AgAVFRWVCAkVBQgJBQkFZbUFCCkVBQYgAAjFbiASYE",
    "deposit": 10,
    "vm_version": 3
  }
}
```

### Contract create from on-chain contract

 * **method:** `channels.update.new_contract_from_onchain`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | contract | contract id | on-chain contract id | Yes |
  | call\_data | call data | call data for contract creation | Yes |
  | deposit | integer | contract creation deposit | Yes |

 * **no response**

#### Example

```javascript
{
  "jsonrpc": "2.0"
  "method": "channels.update.new_contract_from_onchain",
  "params": {
    "call_data": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACC5yVbyizFJqfWYeqUF89obIgnMVzkjQAYrtsG9n5+Z6gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAnHQYrA==",
    "contract": "ct_26WLX27MDif5WcqBXX8ndnQ2TQQvLgZ4TfL299BmteioZsJYER",
    "deposit": 10,
  },
}
```

### Contract call

 * **method:** `channels.update.call_contract`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | contract | contract id | contract to call | Yes |
  | call\_data | call data | call data | Yes |
  | abi\_version | integer | call abi version | Yes |
  | amount | integer | amount of tokens to transfer to contract | Yes |

 * **no response**

#### Example

```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.update.call_contract",
  "params": {
    "abi_version": 1,
    "amount": 0,
    "call_data": "cb_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACBo8mdjOP9QiDmrpHdJ7/qL6H7yhPIH+z2ZmHAc1TiHxQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACo7dbVl",
    "contract": "ct_2Yy7TpPUs7SCm9jkCz7vz3nkb18zs78vcuVQGbgjRaWQNTWpm5"
  }
}
```

## Generic message
Roles:
 * Sender
 * Receiver

### Sender send message
 * **method:** `channels.message`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | to | string | Receiver's address | Yes |
  | info | string | Message body | Yes |

#### Example
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

### Receiver receives message
 * **method:** `channels.message`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | message | message object | Message | Yes |

 message object:

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | channel_id | string | channel id| Yes |
  | from | string | Sender's address | Yes |
  | to | string | Receiver's address | Yes |
  | info | string | Message body | Yes |

#### Example
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

## Close mutual
Roles:
 * Closer
 * Acknowledger

### Closer initiate mutual close
 * **method:** `channels.shutdown`

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.shutdown",
  "params": {}
}
```

### Closer receives mutual close
 * **method:** `channels.sign.shutdown_sign`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_close_mutual` transaction | Yes |

#### Example
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

### Closer returns signed mutual close
 * **method:** `channels.shutdown_sign`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_close_mutual` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.shutdown_sign",
  "params": {
    "tx": "tx_+KcLAfhCuEBjFLnPgoPp15I2oYzgPjrQJQyqvg1O1EimmGosw9jrNw2jTfaZXzSleSqPJ7F5T/dqLUSmqN5Py7ZlWQFKn4IFuF/4XTUBoQY/KKbrcmIHVGiACvxWdOMPb8t4vp8YuHJJaGqCvt7teKEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGNpHWr7//hhtI61fgAQCGEjCc5UAAA9iOZm4="
  }
}
```

### Acknowledger receives mutual close
 * **method:** `channels.sign.shutdown_sign_ack`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | unsigned `channel_close_mutual` transaction | Yes |

#### Example
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

### Acknowledger returns signed mutual close
 * **method:** `channels.shutdown_sign_ack`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | signed `channel_close_mutual` transaction | Yes |

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.shutdown_sign_ack",
  "params": {
    "tx": "tx_+KcLAfhCuECzm2E8CsyzKvbO6zx8QHMnqX2lorQfH/t+FE7SQsf7tAL32BbqyFMox0su5RN9nriuXW1zdSCBkNvtKGDyVukCuF/4XTUBoQY/KKbrcmIHVGiACvxWdOMPb8t4vp8YuHJJaGqCvt7teKEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2aGNpHWr7//hhtI61fgAQCGEjCc5UAAA3BBwE8="
  }
}
```

## On-chain transactions
 * **method:** `channels.on_chain_tx`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | tx | string | co-signed transaction that is posted on-chain | Yes |

#### Example
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

## Leave
Roles:
 * Leaver
 * Acknowledger

### Leaver initiates leave
 * **method:** `channels.leave`

#### Example
```javascript
{
  "jsonrpc": "2.0",
  "method": "channels.leave",
  "params": {}
}
```

#### Leaver and Acknowledger inform their clients
```javascript
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

## Info messages

### Info
 * **method:** `channels.info`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | event | string | event description | Yes |

#### Example
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

### Latest state
 * **method:** `channels.get.offchain_state`

#### Response
 | Field name | Value |
 | ---- | ---- |
 | `trees` | channel state trees |
 | `calls` | channel call state tree |
 | `half_signed_tx` | channel latest half signed tx or `''` if equal to latest signed tx |
 | `signed_tx` | channel latest fully signed tx or `''` if not available |

#### Example

##### Request
```javascript
{
  "id": -576460752303423485,
  "jsonrpc": "2.0",
  "method": "channels.get.offchain_state",
  "params": {}
}
```

##### Response
```javascript
{
  "channel_id": "ch_2TPA12qYnQUE6359Vmg55SD5wMvKUQxZA9wWChWSsRkFvvYAUd",
  "id": -576460752303423485,
  "jsonrpc": "2.0",
  "result": {
    "calls": "cs_yYICbgGEwz8BwHtqgWY=",
    "half_signed_tx": "",
    "signed_tx": "tx_+QEhCwH4hLhAFXwNaSWq75TZB6MqjXb237eg1HXAwtRJjHZSYxRYBAvvvuLHEZiGLiCJ1rrQpPdUf1iez8yMwBo3ufn+zQCXA7hA15Jy3irdXz/9xhNWcsbNAZGKh6zYtA1Q4jG8Hd7SWESzPCKGI2mcD1CRRo1VU/6wtJn4g2pYjdo1fu464x2sCLiX+JU5AaEGv5mAzmFWTbhQbZIllPBKUMvVeYksW98zTz0CISoGPl0C+E24S/hJggI6AaEBsbV3vNMnyznlXmwCa9anShs13mwGUMSuUe+rdZ5BW2ahAWccVUZGSUV1srSU9lFoIXEGY9hIk83S0jYDelTDPu6EAaDVKneBkzhWnEbC2q9FakzUh62kF75skKy/zAal9PdC4a8Rvaw=",
    "trees": "ss_+Ks+AIrJggJtAYTDPwHAismCAm4BhMM/AcCKyYICbwGEwz8BwIrJggJwAYTDPwHAismCAnEBhMM/AcC4cPhuggJyAbho+GY/AfhisO9AAaCxtXe80yfLOeVebAJr1qdKGzXebAZQxK5R76t1nkFbZovKCgEAhj+qJSJf/rDvQAGgZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oSLygoBAIYkYTnKgAJXk16b"
  },
  "version": 1
}
```

### Latest contract state
 * **method:** `channels.get.contract`
 * **payload:**

  | Name | Type | Description | Required |
  | ---- | ---- | ----------- | -------- |
  | pubkey | string | requested contract pubkey | Yes |

#### Response
 | Field name | Value |
 | ---- | ---- |
 | `contract` | object with contract details |
 | `contract->id` | contract id (equals to the requested pubkey) |
 | `contract->owner_id` | contract owner id |
 | `contract->vm_version` | contract vm version (integer) |
 | `contract->abi_version` | contract ABI version (integer) |
 | `contract->active` | "is contract active?" boolean |
 | `contract->referrer_ids` | referrer ids list |
 | `contract->deposit` | contract deposit |
 | `contract_state` | object with contract state |

#### Example

##### Request
```javascript
{
  "id": -576460752303423453,
  "jsonrpc": "2.0",
  "method": "channels.get.contract",
  "params": {
    "pubkey": "ct_uBX2jBr5bPEzD1uGFmV4i7JrGLtpEeLqPU29HvkZCv5iHcY4M"
  }
}
```

##### Response
```javascript
{
  "channel_id": "ch_2WDx2uyKSNAMSEb11eczNNtSMyRLNfhFAUwwP73xDk4cngy9u9",
  "id": -576460752303423378,
  "jsonrpc": "2.0",
  "result": {
    "contract": {
      "abi_version": 1,
      "active": true,
      "deposit": 10,
      "id": "ct_MkcRwxFzpHUUgcxM9fJ6YSPwoqriqZkrD7UwV2oQd1eCaniw5",
      "owner_id": "ak_nQpnNuBPQwibGpSJmjAah6r3ktAB7pG9JHuaGWHgLKxaKqEvC",
      "referrer_ids": [],
      "vm_version": 3
    },
    "contract_state": {
      "ck_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAFSSB3aW4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACG5Hw1": "cv_ZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oQn68jh",
      "ck_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJbm8sIEkgd2luAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD8NuDd": "cv_ZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oQn68jh",
      "ck_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMSSBjbGFpbSB0aGlzAAAAAAAAAAAAAAAAAAAAAAAAAACEEDl2": "cv_ZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oQn68jh",
      "ck_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAXb3RoZXIgcmVhc29uYWJsZSB0aGluZ3kAAAAAAAAAAACz7RfH": "cv_ZxxVRkZJRXWytJT2UWghcQZj2EiTzdLSNgN6VMM+7oQn68jh",
      "ck_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAArMtts": "cv_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAPE19M0=",
      "ck_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAARfjTDm": "cv_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJhxCx4=",
      "ck_ABQG4Fg=": "cv_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACDIq10LA12xQcUbe3+xpb7TFHBPbFVOWLUaz7/TKOGKmgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJEZpbGwgbWUgaW4gd2l0aCBzb21ldGhpbmcgcmVhc29uYWJsZQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADHrnyz",
      "ck_AZwSz9w=": "cv_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAWD//////////////////////////////////////////wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAcAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAB4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyVHQH",
      "ck_AhzDreo=": "cv_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAARfjTDm"
    }
  },
  "version": 1
}
```
