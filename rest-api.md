<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*
<br/>BitKings Exchange API has been elaborated using Binance *[API documentation](https://github.com/binance-exchange/binance-official-api-docs)* as the main source of reference 

  - [General API Information](#general-api-information)
  - [HTTP Return Codes](#http-return-codes)
  - [Error Codes](#error-codes)
  - [General Information on Endpoints](#general-information-on-endpoints)
- [Endpoint security type](#endpoint-security-type)
- [SIGNED (TRADE and USER_DATA) Endpoint security](#signed-trade-and-user_data-endpoint-security)
  - [Timing security](#timing-security)
  - [SIGNED Endpoint Examples for POST /api/v3/order](#signed-endpoint-examples-for-post-apiv3order)
    - [Example 1: As a request body](#example-1-as-a-request-body)
    - [Example 2: As a query string](#example-2-as-a-query-string)
    - [Example 3: Mixed query string and request body](#example-3-mixed-query-string-and-request-body)
- [Public API Endpoints](#public-api-endpoints)
  - [Terminology](#terminology)
  - [ENUM definitions](#enum-definitions)
  - [General endpoints](#general-endpoints)
    - [Test connectivity](#test-connectivity)
    - [Check server time](#check-server-time)
    - [Exchange information](#exchange-information)
  - [Market Data endpoints](#market-data-endpoints)
    - [Order book](#order-book)
    - [Recent trades list](#recent-trades-list)
    - [Old trade lookup (MARKET_DATA)](#old-trade-lookup-market_data)
    - [Compressed/Aggregate trades list](#compressedaggregate-trades-list)
    - [Kline/Candlestick data](#klinecandlestick-data)
    - [Current average price](#current-average-price)
    - [24hr ticker price change statistics](#24hr-ticker-price-change-statistics)
    - [Symbol price ticker](#symbol-price-ticker)
    - [Symbol order book ticker](#symbol-order-book-ticker)
  - [Account endpoints](#account-endpoints)
    - [New order  (TRADE)](#new-order--trade)
    - [Test new order (TRADE)](#test-new-order-trade)
    - [Query order (USER_DATA)](#query-order-user_data)
    - [Cancel order (TRADE)](#cancel-order-trade)
    - [Cancel All Open Orders on a Symbol (TRADE)](#cancel-all-open-orders-on-a-symbol-trade)
    - [Current open orders (USER_DATA)](#current-open-orders-user_data)
    - [All orders (USER_DATA)](#all-orders-user_data)
    - [Account information (USER_DATA)](#account-information-user_data)
    - [Account trade list (USER_DATA)](#account-trade-list-user_data)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Public Rest API for BitKings (2020-09-08)

## General API Information
* The base endpoint is: **https://api.bitkings.com**
* All endpoints return either a JSON object or array.
* Data is returned in **ascending** order. Oldest first, newest last.
* All time and timestamp related fields are in **milliseconds**.

## HTTP Return Codes

* HTTP `4XX` return codes are used for malformed requests;
  the issue is on the sender's side.
* HTTP `403` return code is used when the WAF Limit (Web Application Firewall) has been violated.
* HTTP `429` return code is used when breaking a request rate limit.
* HTTP `418` return code is used when an IP has been auto-banned for continuing to send requests after receiving `429` codes.
* HTTP `5XX` return codes are used for internal errors; the issue is on
  BitKing's side.
  It is important to **NOT** treat this as a failure operation; the execution status is
  **UNKNOWN** and could have been a success.


## Error Codes
* Any endpoint can return an ERROR

Sample Payload below:
```javascript
{
  code: -1002,
  msg: 'Unauthorized request - signature is incorrect'
}
```
* Specific error codes and messages are defined in [Errors Codes](./errors.md).

## General Information on Endpoints
* For `GET` endpoints, parameters must be sent as a `query string`.
* For `POST`, `PUT`, and `DELETE` endpoints, the parameters may be sent as a
  `query string` or in the `request body` with content type
  `application/x-www-form-urlencoded`. You may mix parameters between both the
  `query string` and `request body` if you wish to do so.
* Parameters may be sent in any order.
* If a parameter sent in both the `query string` and `request body`, the
  `query string` parameter will be used.


# Endpoint security type
* Each endpoint has a security type that determines the how you will
  interact with it. This is stated next to the NAME of the endpoint.
    * If no security type is stated, assume the security type is NONE.
* API-keys are passed into the Rest API via the `X-MBX-APIKEY`
  header.
* API-keys and secret-keys **are case sensitive**.
* API-keys can be configured to only access certain types of secure endpoints.
 For example, one API-key could be used for TRADE only, while another API-key
 can access everything except for TRADE routes.
* By default, API-keys can access all secure routes.

Security Type | Description
------------ | ------------
NONE | Endpoint can be accessed freely.
TRADE | Endpoint requires sending a valid API-Key and signature.
USER_DATA | Endpoint requires sending a valid API-Key and signature.
USER_STREAM | Endpoint requires sending a valid API-Key.
MARKET_DATA | Endpoint requires sending a valid API-Key.


* `TRADE` and `USER_DATA` endpoints are `SIGNED` endpoints.

# SIGNED (TRADE and USER_DATA) Endpoint security
* `SIGNED` endpoints require an additional parameter, `signature`, to be
  sent in the  `query string` or `request body`.
* Endpoints use `HMAC SHA256` signatures. The `HMAC SHA256 signature` is a keyed `HMAC SHA256` operation.
  Use your `secretKey` as the key and `totalParams` as the value for the HMAC operation.
* The `signature` is **not case sensitive**.
* `totalParams` is defined as the `query string` concatenated with the
  `request body`.

## Timing security
* A `SIGNED` endpoint also requires a parameter, `timestamp`, to be sent which
  should be the millisecond timestamp of when the request was created and sent.
* An additional parameter, `recvWindow`, may be sent to specify the number of
  milliseconds after `timestamp` the request is valid for. If `recvWindow`
  is not sent, **it defaults to 5000**.
* The logic is as follows:
  ```javascript
  if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow) {
    // process request
  } else {
    // reject request
  }
  ```

**Serious trading is about timing.** Networks can be unstable and unreliable,
which can lead to requests taking varying amounts of time to reach the
servers. With `recvWindow`, you can specify that the request must be
processed within a certain number of milliseconds or be rejected by the
server.


**It is recommended to use a small recvWindow of 5000 or less! The max cannot go beyond 60,000!**


## SIGNED Endpoint Examples for POST /api/v3/order
Here is a step-by-step example of how to send a valid signed payload from the
Linux command line using `echo`, `openssl`, and `curl`.

Key | Value
------------ | ------------
apiKey | vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A
secretKey | NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j


Parameter | Value
------------ | ------------
symbol | USDTBTK
side | BUY
type | LIMIT
timeInForce | GTC
quantity | 1
price | 0.1
recvWindow | 5000
timestamp | 1499827319559

### Example 1: As a request body
* **requestBody:** symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559
* **HMAC SHA256 signature:**

    ```
    [linux]$ echo -n "symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
    ```


* **curl command:**

    ```
    (HMAC SHA256)
    [linux]$ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.bitkings.com/api/v1/order' -d 'symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
    ```

### Example 2: As a query string
* **queryString:** symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559
* **HMAC SHA256 signature:**

    ```
    [linux]$ echo -n "symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
    ```


* **curl command:**

    ```
    (HMAC SHA256)
    [linux]$ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.bitkings.com/api/v1/order?symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
    ```

### Example 3: Mixed query string and request body
* **queryString:** symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC
* **requestBody:** quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559
* **HMAC SHA256 signature:**

    ```
    [linux]$ echo -n "symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTCquantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= 0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77
    ```


* **curl command:**

    ```
    (HMAC SHA256)
    [linux]$ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.bitkings.com/api/v1/order?symbol=BTKUSDT&side=BUY&type=LIMIT&timeInForce=GTC' -d 'quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77'
    ```

Note that the signature is different in example 3.
There is no & between "GTC" and "quantity=1".

# Public API Endpoints
### Terminology

These terms will be used throughout the documentation, so it is recommended especially for new users to read to help their understanding of the API.

* `base asset` refers to the asset that is the `quantity` of a symbol. For the symbol BTKUSDT, BTK would be the `base asset`.
* `quote asset` refers to the asset that is the `price` of a symbol. For the symbol BTKUSDT, USDT would be the `quote asset`.


## ENUM definitions
**Symbol status (status):**

* PRE_TRADING
* TRADING
* POST_TRADING
* END_OF_DAY
* HALT
* AUCTION_MATCH
* BREAK

**Symbol type:**

* SPOT

**Order status (status):**

Status | Description
-----------| --------------
`NEW` | The order has been accepted by the engine.
`PARTIALLY_FILLED`| A part of the order has been filled.
`FILLED` | The order has been completed.
`CANCELED` | The order has been canceled by the user.
` PENDING_CANCEL` | Currently unused
`REJECTED`       | The order was not accepted by the engine and not processed.
`EXPIRED` | The order was canceled according to the order type's rules (e.g. LIMIT FOK orders with no fill, LIMIT IOC or MARKET orders that partially fill) <br> or by the exchange, (e.g. orders canceled during liquidation, orders canceled during maintenance)


**Order types (orderTypes, type):**

More information on how the order types definitions can be found here: [Types of Orders](https://www.bitkings.com/en/support/articles/360033779452-Types-of-Order)

* LIMIT
* MARKET
* STOP_LOSS
* STOP_LOSS_LIMIT
* TAKE_PROFIT
* TAKE_PROFIT_LIMIT
* LIMIT_MAKER

**Order Response Type (newOrderRespType):**

* ACK
* RESULT
* FULL

**Order side (side):**

* BUY
* SELL

**Time in force (timeInForce):**

This sets how long an order will be active before expiration.

Status | Description
-----------| --------------
`GTC` | Good Til Canceled <br> An order will be on the book unless the order is canceled. 
`IOC` | Immediate Or Cancel <br> An order will try to fill the order as much as it can before the order expires.
`FOK`| Fill or Kill <br> An order will expire if the full order cannot be filled upon execution.

**Kline/Candlestick chart intervals:**

m -> minutes; h -> hours; d -> days; w -> weeks; M -> months

* 1m
* 3m
* 5m
* 15m
* 30m
* 1h
* 2h
* 4h
* 6h
* 8h
* 12h
* 1d
* 3d
* 1w
* 1M

**Rate limiters (rateLimitType)**
* REQUEST_WEIGHT

    ```json
    {
      "rateLimitType": "REQUEST_WEIGHT",
      "interval": "MINUTE",
      "intervalNum": 1,
      "limit": 1200
    }
    ```

* ORDERS

    ```json
    {
      "rateLimitType": "ORDERS",
      "interval": "SECOND",
      "intervalNum": 1,
      "limit": 10
    }
    ```
* RAW_REQUESTS

    ```json
    {
      "rateLimitType": "RAW_REQUESTS",
      "interval": "MINUTE",
      "intervalNum": 5,
      "limit": 5000
    }
    ```

**Rate limit intervals (interval)**

* SECOND
* MINUTE
* DAY

## General endpoints
### Test connectivity
```
GET /api/v3/ping
```
Test connectivity to the Rest API.

**Weight:**
1

**Parameters:**
NONE

**Response:**
```javascript
{}
```

### Check server time
```
GET /api/v3/time
```
Test connectivity to the Rest API and get the current server time.

**Weight:**
1

**Parameters:**
NONE

**Response:**
```javascript
{
  serverTime: 1599657467533
}
```

### Exchange information
```
GET /api/v3/exchangeInfo
```
Current exchange trading rules and symbol information

**Weight:**
1

**Parameters:**
NONE

**Response:**
```javascript
200 {
  timezone: 'UTC',
  serverTime: 1599657344565,
  rateLimits: [],
  exchangeFilters: [],
  symbols: [
    {
      symbol: 'USDTETH',
      status: 'TRADING',
      baseAsset: 'ETH',
      baseAssetPrecision: 8,
      quoteAsset: 'USDT',
      quotePrecision: 8,
      baseCommissionPrecision: 8,
      quoteCommissionPrecision: 8,
      orderTypes: [Array],
      filters: []
    },
    {
      symbol: 'USDTBTK',
      status: 'TRADING',
      baseAsset: 'BTK',
      baseAssetPrecision: 8,
      quoteAsset: 'USDT',
      quotePrecision: 8,
      baseCommissionPrecision: 8,
      quoteCommissionPrecision: 8,
      orderTypes: [Array],
      filters: []
    },
    {
      symbol: 'USDTXAMP',
      status: 'TRADING',
      baseAsset: 'XAMP',
      baseAssetPrecision: 8,
      quoteAsset: 'USDT',
      quotePrecision: 8,
      baseCommissionPrecision: 8,
      quoteCommissionPrecision: 8,
      orderTypes: [Array],
      filters: []
    }
  ]
}
```

## Market Data endpoints
### Order book
```
GET /api/v3/depth
```

**Weight:**
Adjusted based on the limit:


Limit | Weight
------------ | ------------
5, 10, 20, 50, 100 | 1
500 | 5
1000 | 10
5000| 50


**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 100; max 5000. Valid limits:[5, 10, 20, 50, 100, 500, 1000, 5000]

**Response:**
```javascript
{
  lastUpdateId: 1,
  bids: [
    [
      0.000274,     // PRICE
      15000         // QTY
    ]
  ],
  asks: [
    [
      0.000274,
      5000
    ]
  ]
}
```


### Recent trades list
```
GET /api/v3/trades
```
Get recent trades (up to last 500).

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.

**Response:**
```javascript
[
  {
    id: 12,
    price: 0.00029,
    qty: 5000,
    time: 1599476027866,
    isBuyerMaker: true
  }
]
```

### Old trade lookup (MARKET_DATA)
```
GET /api/v3/historicalTrades
```
Get older trades.

**Weight:**
5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.
fromId | LONG | NO | TradeId to fetch from. Default gets most recent trades.

**Response:**
```javascript
[
  {
    id: 12,
    price: 0.00029,
    qty: 5000,
    time: 1599476027866,
    isBuyerMaker: true
  }
]
```

### Compressed/Aggregate trades list
```
GET /api/v3/aggTrades
```
Get compressed, aggregate trades. Trades that fill at the time, from the same taker order, with the same price will have the quantity aggregated.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
fromId | LONG | NO | ID to get aggregate trades from INCLUSIVE.
startTime | LONG | NO | Timestamp in ms to get aggregate trades from INCLUSIVE.
endTime | LONG | NO | Timestamp in ms to get aggregate trades until INCLUSIVE.
limit | INT | NO | Default 500; max 1000.

* If both startTime and endTime are sent, time between startTime and endTime must be less than 1 hour.
* If fromId, startTime, and endTime are not sent, the most recent aggregate trades will be returned.

**Response:**
```javascript
[
  {
    "a": 26129,         // Aggregate tradeId
    "p": "0.01633102",  // Price
    "q": "4.70443515",  // Quantity
    "f": 27781,         // First tradeId
    "l": 27781,         // Last tradeId
    "T": 1498793709153, // Timestamp
    "m": true,          // Was the buyer the maker?
    "M": true           // Was the trade the best price match?
  }
]
```

### Kline/Candlestick data
```
GET /api/v3/klines
```
Kline/candlestick bars for a symbol.
Klines are uniquely identified by their open time.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
interval | ENUM | YES |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.

* If startTime and endTime are not sent, the most recent klines are returned.

**Response:**
```javascript
[
  [
    1499040000000,      // Open time
    "0.01634790",       // Open
    "0.80000000",       // High
    "0.01575800",       // Low
    "0.01577100",       // Close
    "148976.11427815",  // Volume
    1499644799999,      // Close time
    "2434.19055334",    // Quote asset volume
    308,                // Number of trades
    "1756.87402397",    // Taker buy base asset volume
    "28.46694368",      // Taker buy quote asset volume
    "17928899.62484339" // Ignore.
  ]
]
```


### Current average price
Current average price for a symbol.
```
GET /api/v3/avgPrice
```
**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |


**Response:**
```javascript
{
  mins: 5,
  price: 0.0003
}
```


### 24hr ticker price change statistics
```
GET /api/v3/ticker/24hr
```
24 hour rolling window price change statistics. **Careful** when accessing this with no symbol.

**Weight:**
1 for a single symbol; **40** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* If the symbol is not sent, tickers for all symbols will be returned in an array.

**Response:**
```javascript
{
  symbol: 'USDTBTK',
  lastPrice: 0.0003,
  bidPrice: 0.000299,
  askPrice: 0.0003
}
```


### Symbol price ticker
```
GET /api/v3/ticker/price
```
Latest price for a symbol or symbols.

**Weight:**
1 for a single symbol; **2** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* If the symbol is not sent, prices for all symbols will be returned in an array.

**Response:**
```javascript
{
  symbol: 'USDTBTK',
  price: 0.0003
}
```


### Symbol order book ticker
```
GET /api/v3/ticker/bookTicker
```
Best price/qty on the order book for a symbol or symbols.

**Weight:**
1 for a single symbol; **2** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* If the symbol is not sent, bookTickers for all symbols will be returned in an array.

**Response:**
```javascript
{
    symbol: 'BTKUSDT',
    bidPrice: 0.000299,
    bidQty: 15000,
    askPrice: 0.0003,
    askQty: 15000
}
```


## Account endpoints
### New order  (TRADE)
```
POST /api/v3/order  (HMAC SHA256)
```
Send in a new order.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
side | ENUM | YES |
type | ENUM | YES |
timeInForce | ENUM | NO |
quantity | DECIMAL | NO |
quoteOrderQty|DECIMAL|NO|
price | DECIMAL | NO |
newClientOrderId | STRING | NO | A unique id among open orders. Automatically generated if not sent.<br> Orders with the same `newClientOrderID` can be accepted only when the previous one is filled, otherwise the order will be rejected.
stopPrice | DECIMAL | NO | Used with `STOP_LOSS`, `STOP_LOSS_LIMIT`, `TAKE_PROFIT`, and `TAKE_PROFIT_LIMIT` orders.
newOrderRespType | ENUM | NO | Set the response JSON. `ACK`, `RESULT`, or `FULL`; `MARKET` and `LIMIT` order types default to `FULL`, all other orders default to `ACK`.
recvWindow | LONG | NO |The value cannot be greater than ```60000```
timestamp | LONG | YES |

Some additional mandatory parameters based on order `type`:

Type | Additional mandatory parameters | Additional Information
------------ | ------------| ------
`LIMIT` | `timeInForce`, `quantity`, `price`| 
`MARKET` | `quantity` or `quoteOrderQty`| `MARKET` orders using the `quantity` field specifies the amount of the `base asset` the user wants to buy or sell at the market price. <br> E.g. MARKET order on BTCUSDT will specify how much BTC the user is buying or selling. <br><br> `MARKET` orders using `quoteOrderQty` specifies the amount the user wants to spend (when buying) or receive (when selling) the `quote` asset; the correct `quantity` will be determined based on the market liquidity and `quoteOrderQty`. <br> E.g. Using the symbol BTCUSDT: <br> `BUY` side, the order will buy as many BTC as `quoteOrderQty` USDT can. <br> `SELL` side, the order will sell as much BTC needed to receive `quoteOrderQty` USDT.
`STOP_LOSS` | `quantity`, `stopPrice`| This will execute a `MARKET` order when the `stopPrice` is reached.
`STOP_LOSS_LIMIT` | `timeInForce`, `quantity`,  `price`, `stopPrice` 
`TAKE_PROFIT` | `quantity`, `stopPrice`| This will execute a `MARKET` order when the `stopPrice` is reached.
`TAKE_PROFIT_LIMIT` | `timeInForce`, `quantity`, `price`, `stopPrice` | 
`LIMIT_MAKER` | `quantity`, `price`| This is a `LIMIT` order that will be rejected if the order immediately matches and trades as a taker. <br> This is also known as a POST-ONLY order. 

Other info:

* `MARKET` orders using `quoteOrderQty` will not break `LOT_SIZE` filter rules; the order will execute a `quantity` that will have the notional value as close as possible to `quoteOrderQty`.
Trigger order price rules against market price for both MARKET and LIMIT versions:

* Price above market price: `STOP_LOSS` `BUY`, `TAKE_PROFIT` `SELL`
* Price below market price: `STOP_LOSS` `SELL`, `TAKE_PROFIT` `BUY`

**Response ACK:**
```javascript
{
  "symbol": "ETHUSDT",
  "orderId": 28,
  "orderListId": -1, 
  "clientOrderId": "6gCrw2kRUAF9CvJDGP16IP",
  "transactTime": 1507725176595
}
```

**Response RESULT:**
```javascript
{
  "symbol": "BTKUSDT",
  "orderId": 28,
  "orderListId": -1, 
  "clientOrderId": "6gCrw2kRUAF9CvJDGP16IP",
  "transactTime": 1507725176595,
  "price": "0.00000000",
  "origQty": "10.00000000",
  "executedQty": "10.00000000",
  "cummulativeQuoteQty": "10.00000000",
  "status": "FILLED",
  "timeInForce": "GTC",
  "type": "MARKET",
  "side": "SELL"
}
```

**Response FULL:**
```javascript
{
  "symbol": "BTCUSDT",
  "orderId": 28,
  "orderListId": -1,
  "clientOrderId": "6gCrw2kRUAF9CvJDGP16IP",
  "transactTime": 1507725176595,
  "price": "0.00000000",
  "origQty": "10.00000000",
  "executedQty": "10.00000000",
  "cummulativeQuoteQty": "10.00000000",
  "status": "FILLED",
  "timeInForce": "GTC",
  "type": "MARKET",
  "side": "SELL",
  "fills": [
    {
      "price": "4000.00000000",
      "qty": "1.00000000",
      "commission": "4.00000000",
      "commissionAsset": "USDT"
    },
    {
      "price": "3999.00000000",
      "qty": "5.00000000",
      "commission": "19.99500000",
      "commissionAsset": "USDT"
    },
    {
      "price": "3998.00000000",
      "qty": "2.00000000",
      "commission": "7.99600000",
      "commissionAsset": "USDT"
    },
    {
      "price": "3997.00000000",
      "qty": "1.00000000",
      "commission": "3.99700000",
      "commissionAsset": "USDT"
    },
    {
      "price": "3995.00000000",
      "qty": "1.00000000",
      "commission": "3.99500000",
      "commissionAsset": "USDT"
    }
  ]
}
```

### Test new order (TRADE)
```
POST /api/v3/order/test (HMAC SHA256)
```
Test new order creation and signature/recvWindow long.
Creates and validates a new order but does not send it into the matching engine.

**Weight:**
1

**Parameters:**

Same as `POST /api/v3/order`


**Response:**
```javascript
{}
```

### Query order (USER_DATA)
```
GET /api/v3/order (HMAC SHA256)
```
Check an order's status.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
recvWindow | LONG | NO | The value cannot be greater than ```60000```
timestamp | LONG | YES |

Notes:
* Either `orderId` or `origClientOrderId` must be sent.
* For some historical orders `cummulativeQuoteQty` will be < 0, meaning the data is not available at this time.


**Response:**
```javascript
{
  "symbol": "LTCBTC",
  "orderId": 1,
  "orderListId": -1
  "clientOrderId": "myOrder1",
  "price": "0.1",
  "origQty": "1.0",
  "executedQty": "0.0",
  "cummulativeQuoteQty": "0.0",
  "status": "NEW",
  "timeInForce": "GTC",
  "type": "LIMIT",
  "side": "BUY",
  "stopPrice": "0.0",
  "time": 1499827319559,
  "updateTime": 1499827319559,
  "isWorking": true,
  "origQuoteOrderQty": "0.000000"
}
```

### Cancel order (TRADE)
```
DELETE /api/v3/order  (HMAC SHA256)
```
Cancel an active order.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
newClientOrderId | STRING | NO |  Used to uniquely identify this cancel. Automatically generated by default.
recvWindow | LONG | NO | The value cannot be greater than ```60000```
timestamp | LONG | YES |

Either `orderId` or `origClientOrderId` must be sent.

**Response:**
```javascript
{
  symbol: 'USDTETH',
  orderId: 35791,
  price: 700,
  origQty: 0.03,
  executedQty: 0,
  status: 'CANCELLED',
  type: 'LIMIT',
  side: 'SELL',
  time: 1599646767000
}
```

### Cancel All Open Orders on a Symbol (TRADE)
```
DELETE /api/v3/openOrders (HMAC SHA256)
```
Cancels all active orders on a symbol.

**Weight**
1

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
recvWindow | LONG | NO | The value cannot be greater than ```60000```
timestamp | LONG | YES |

**Response**
```javascript
[
  {
  symbol: 'USDTETH',
  orderId: 35791,
  price: 700,
  origQty: 0.03,
  executedQty: 0,
  status: 'CANCELLED',
  type: 'LIMIT',
  side: 'SELL',
  time: 1599646767000
  },
]
```

### Current open orders (USER_DATA)
```
GET /api/v3/openOrders  (HMAC SHA256)
```
Get all open orders on a symbol. **Careful** when accessing this with no symbol.

**Weight:**
1 for a single symbol; **40** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |
recvWindow | LONG | NO | The value cannot be greater than ```60000```
timestamp | LONG | YES |

* If the symbol is not sent, orders for all symbols will be returned in an array.

**Response:**
```javascript
[
  {
    symbol: 'USDTBTK',
    orderId: 3453,
    price: 0.0003,
    origQty: 4984.38,
    executedQty: 0,
    status: 'NEW',
    type: 'LIMIT',
    side: 'SELL',
    time: 1599558116000
  }
]
```

### All orders (USER_DATA)
```
GET /api/v3/allOrders (HMAC SHA256)
```
Get all account orders; active, canceled, or filled.

**Weight:**
5 with symbol

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO | The value cannot be greater than ```60000```
timestamp | LONG | YES |

**Notes:**
* If `orderId` is set, it will get orders >= that `orderId`. Otherwise most recent orders are returned.
* For some historical orders `cummulativeQuoteQty` will be < 0, meaning the data is not available at this time.

**Response:**
```javascript
[
  {
    "symbol": "LTCBTC",
    "orderId": 1,
    "orderListId": -1,
    "clientOrderId": "myOrder1",
    "price": "0.1",
    "origQty": "1.0",
    "executedQty": "0.0",
    "cummulativeQuoteQty": "0.0",
    "status": "NEW",
    "timeInForce": "GTC",
    "type": "LIMIT",
    "side": "BUY",
    "stopPrice": "0.0",
    "time": 1499827319559,
    "updateTime": 1499827319559,
    "isWorking": true,
    "origQuoteOrderQty": "0.000000"
  }
]
```


### Account information (USER_DATA)
```
GET /api/v3/account (HMAC SHA256)
```
Get current account information.

**Weight:**
5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO | The value cannot be greater than ```60000```
timestamp | LONG | YES |

**Response:**
```javascript
{
  canTrade: true,
  canWithdraw: true,
  canDeposit: true,
  updateTime: 1599562080318,
  balances: [
    { asset: 'XAMP', free: 0, locked: 0 },
    { asset: 'XRP', free: 0, locked: 0 },
    { asset: 'LTC', free: 0, locked: 0 },
    { asset: 'USDT', free: 0.007990005606998807, locked: 0 },
    { asset: 'BTK', free: 4984.380581900001, locked: 0 },
    { asset: 'TRX', free: 0, locked: 0 },
    { asset: 'ETH', free: 0.03109684499999997, locked: 0 },
    { asset: 'BTC', free: 0, locked: 0 }
  ],
  makerCommission: null,
  takerCommission: null
  ]
}
```

### Account trade list (USER_DATA)
```
GET /api/v3/myTrades  (HMAC SHA256)
```
Get trades for a specific account and symbol.

**Weight:**
5 with symbol

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
startTime | LONG | NO |
endTime | LONG | NO |
fromId | LONG | NO | TradeId to fetch from. Default gets most recent trades.
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO | The value cannot be greater than ```60000```
timestamp | LONG | YES |

**Notes:**
* If `fromId` is set, it will get orders >= that `fromId`.
Otherwise most recent orders are returned.

**Response:**
```javascript
[
  {
    symbol: 'USDTBTK',
    id: 22,
    orderId: 124,
    price: 0.00029,
    qty: 4984.3805819,
    commission: 0,
    commissionAsset: 'USDT',
    time: 1599493279836,
    isBuyer: 'Y',
    isMaker: 'Y'
  }
]
```
