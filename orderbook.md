# Degens Orderbook API

This document describes the low-level orderbook websocket protocol.

* Example implementation: [OrderbookClient.js](https://github.com/degensprotocol/degens-jslib/blob/master/OrderbookClient.js)
* Market maker code that uses this implementation: [mm.js](https://github.com/degensprotocol/degens-mm)

## URL

For mainnet, a websocket connection is made to the following URL:

    wss://degens.com/ws

## Framing

Every message, in each direction, consists of 3 items, separated by `|` characters:

* The message type: Indicates what type of message this is
* The message id: A sequence identifer
* The message body: JSON

The reason the type and id are not inside the JSON is so that routing can be performed by examining them without necessarily needing to parse the body.

For example, the message type `hello` must be sent first by the client to exchange information about the connection:

    hello|1|{"version":"client version 1.0"}

The server would then respond with:

    rf|1|{"config":{...}}

Note that any `id` could have been chosen instead of `1`. It is the client's job to keep them unique per request.

The response type above was `rf`. There are 3 possible response types:

* `rf`: Request succeeded and the request is now finished
* `rr`: Request succeeded and there may be more responses in the future (with the same `id`)
* `re`: Error. The request is now finished, and the error reason is in the body (plain text, not JSON)

Since the protocol is asynchronous, `id` is used by the client to match up responses with their corresponding requests.

## Message Types

These are commands sent by the client to the server.

### hello

This is the first operation that a client should send after connecting. It manages the subscriptions for the connection, that is what data the client is interested in receiving.

Request:

    hello|1|{"version":"client version 1.0"}

Response:

Various configuration about the orderbook you have connected to:

    {
      "config": {
        "contract": "0x8888888883585b9a8202Db34D8b09d7252bfc61C",
        "networkId": 1,
        "tokens": {
          "$": {
            "addr": "0x6B175474E89094C44Da98b954EedeAC495271d0F",
            "displayDecimals": 0,
            "dustThreshold": "1000000000000000000",
            "name": "DAI",
            "permitRequestable": true,
            "priority": 1
          },
          "Ξ": {
            "addr": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
            "displayDecimals": 3,
            "dustThreshold": "10000000000000000",
            "name": "WETH",
            "priority": 1
          }
        }
      },
      "instance": "d9e3b6163d454b462d807497fd971a92"
    }


### sub

Registers a subscription. The orderbook will send the current information as well as relevant changes to the client in real-time.

The bodys are in [JSON-Patch](https://tools.ietf.org/html/rfc6902) format

See below for the various subscription types.

Request:

    sub|1|{"to":"vol"}

Response

    rr|1|[{"op":"replace","path":"","value":{"$":[12347764641,59863925657],"Ξ":[64140803,174069704]}}]


### put

Adds one or more orders to the orderbook. The order is [transport packed](https://degensprotocol.github.io/docs/#/protocol?id=transport-packed) and then hex encoded.

The `Order` class in [DegensContractLib.js](https://github.com/degensprotocol/degens-jslib/blob/master/DegensContractLib.js) is an easy way to create orders in the correct format.

Request:

    put|12|[{"order":"0x..."}]

Response:

    rf|12|[{"ok":true}]

### ping

An empty request and response, matched by `id`. Used to keep the websocket connection active since TCP-level SO_KEEPALIVE won't make it through proxies in some situations.

Request:

    ping|22|{}

Response:

    rf|22|{}


## Subscription Types

For each subscription, the server will send sequences of [JSON-Patch](https://tools.ietf.org/html/rfc6902) diffs. The first message will be a `replace` command with `""` as the path, meaning the `value` is the entire initial state. Subsequent patches may just alter portions of the sub-tree. This is done to save bandwidth and also allows client-side immutability optimizations.

### vol

Subscription:

    {"to":"vol"}

State:

    {"$":[12347764641,59863925657],"Ξ":[64140803,174069704]}

This is a map keyed by currency symbol. Each value is an arry of 24h and 7d periods. The units are in millionths of a currency unit.

### account

Subscription:

    {"to":"account","addr":"0x00e21f272a5829c842702d0ba92d99a8727d6207"}

State:

    {
      "bal": {
        "$": [
          "0x0000000000000000000000000000000000000000000000000000000000000000",
          "0x0000000000000000000000000000000000000000000000000000000000000000"
        ],
        "Ξ": [
          "0x0000000000000000000000000000000000000000000000000000000000000000",
          "0x0000000000000000000000000000000000000000000000000000000000000000"
        ]
      },
      "cancelTimestamp": 0
    }

This contains the balance and allowance of the various tokens supported by this orderbook for the `addr` subscribed to.

### events

Gets upcoming, live, or finalized events, and associated orders.

Subscription:

    {"to":"events","search":"","status":"upcoming","limit":12,"page":0}

State:

    {
      "recs": {
        "4555": {
          "event": {
            "kickoff": "1581228000",
            "league": "Womens Olympic Qual",
            "sport": "Soccer",
            "team1": "South Korea Women",
            "team2": "Vietnam Women"
          },
          "eventId": "0x4417f9337b7319c85e8ea7cc84659ede98f95bee8df1bd05c7ba4e638d10d7ef",
          "markets": {
            "0x00d69cf666ce2651f9376d280e1634f299195e9e19fb34e8abe7dc946f9f1bf1": {
              "info": {
                "total": "2.5",
                "type": "total"
              },
              "orders": {
                "1031099": {
                  "amount": "0x0410d586a20a4c0000",
                  "dir": 1,
                  "price": 305543353,
                  "sym": "$"
                },
                "1031100": {
                  "amount": "0x0410d586a20a4c0000",
                  "dir": 0,
                  "price": 234679808,
                  "sym": "$"
                }
              }
            },
          },
          "volume": {
            "$": 254999997
          }
        }
      },
      "more": true
    }

If pagination is used (the `limit` and `page` parameters sent above), then `more` indicates there are more records on the next page.

Note that the record ids (`4555` above) are not globally unique. They are specific to the orderbook you are connected to, and may even change if you reconnect! You should always refer to events using the event ID as described in our protocol docs.


### positions

Subscription:

    {"to":"positions","addr":"0x00e21f272a5829c842702d0ba92d99a8727d6207"}

State:

    {
      "active": 3,
      "claimable": 0,
      "events": {
        "0x1960d63651026d9584adb8bff1b60c51f1b2ef2eaf167b5ef74d9c4818cf436d": {
          "event": {
            "id": "0x1960d63651026d9584adb8bff1b60c51f1b2ef2eaf167b5ef74d9c4818cf436d",
            "kickoff": "1581222600",
            "league": "UFC",
            "sport": "MMA",
            "team1": "Jon Jones",
            "team2": "Dominick Reyes"
          },
          "matches": {
            "0x3f3a6db3d182df3f798f975af8868e13f2e1438875a9438044d4823e49013d52": {
              "market": {
                "type": "1x2_2"
              },
              "tokens": {
                "$": {
                  "avgPrice": 200000000,
                  "pl": "0",
                  "pos": "304599999500000002497"
                }
              }
            }
          }
        },
        "0x9462d927f62e721f4af65384736790c47e659368a435448d6ce0266a550cb9f9": {
          "event": {
            "id": "0x9462d927f62e721f4af65384736790c47e659368a435448d6ce0266a550cb9f9",
            "kickoff": "1580686200",
            "league": "NFL",
            "sport": "American Football",
            "team1": "SF 49ers",
            "team2": "KC Chiefs"
          },
          "matches": {
            "0x16b1f3b311be4592730b5be8d69454d305e3961e7b7f6ca8a5d8ab7315530337": {
              "fin": 0,
              "market": {
                "type": "ml"
              },
              "tokens": {
                "$": {
                  "avgPrice": 476190476,
                  "pl": "0",
                  "pos": "105000000042000000016"
                }
              }
            },
          }
        }
      }
    }

Everything needed to display the position screen: Active and finished matches, your profit-loss (`pl`), average prices, and so on.
