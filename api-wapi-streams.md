# Public WAPI for Naxomart (2018-11-01)
# General API Information
* All endpoints return either a JSON object or array.
* Data is returned in **ascending** order. Oldest first, newest last.
* All time and timestamp related fields are in milliseconds.
* HTTP `4XX` return codes are used for for malformed requests;
  the issue is on the sender's side.
* HTTP `429` return code is used when breaking a request rate limit.
* HTTP `418` return code is used when an IP has been auto-banned for continuing to send requests after receiving `429` codes.
* HTTP `5XX` return codes are used for internal errors; the issue is on
  Naxomart's side.
* HTTP `504` return code is used when the API successfully sent the message
but not get a response within the timeout period.
It is important to **NOT** treat this as a failure; the execution status is
**UNKNOWN** and could have been a success.
* Any endpoint can retun an ERROR; the error payload is as follows:
```javascript
{
  "success": false,
  "msg": "Invalid symbol."
}
```

* Specific error codes and messages defined in another document.
* For `GET` endpoints, parameters must be sent as a `query string`.
* For `POST`, `PUT`, and `DELETE` endpoints, the parameters may be sent as a
  `query string` or in the `request body` with content type
  `application/x-www-form-urlencoded`. You may mix parameters between both the
  `query string` and `request body` if you wish to do so.
* Parameters may be sent in any order.
* If a parameter sent in both the `query string` and `request body`, the
  `query string` parameter will be used.

# LIMITS
* The `/wapi/v3` `rateLimits` array contains objects related to the exchange's `REQUESTS` and `ORDER` rate limits.
* A 429 will be returned when either rather limit is violated.
* Each route has a `weight` which determines for the number of requests each endpoint counts for. Heavier endpoints and endpoints that do operations on multiple symbols will have a heavier `weight`.
* When a 429 is recieved, it's your obligation as an API to back off and not spam the API.
* **Repeatedly violating rate limits and/or failing to back off after receiving 429s will result in an automated IP ban (http status 418).**
* IP bans are tracked and **scale in duration** for repeat offenders, **from 2 minutes to 3 days**.

# Endpoint security type
* Each endpoint has a security type that determines the how you will
  interact with it.
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
* An additional parameter, `recvWindow`, may be sent to specific the number of
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


**Tt recommended to use a small recvWindow of 5000 or less!**


## SIGNED Endpoint Examples for POST /wapi/v3/withdraw.html
Here is a step-by-step example of how to send a vaild signed payload from the
Linux command line using `echo`, `openssl`, and `curl`.

Key | Value
------------ | ------------
apiKey | kmDHN4Hmv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPykhnH4co14y7Ju91du6Nd
secretKey | KH7NtmdSJYdKjVHjA7PZj4M6G2R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fJ87Nl


Parameter | Value
------------ | ------------
asset | ETH
address  |0x001866Ae5B3de6cAa5a51543FD9fB64f524F5478
addressTag | 1 (Secondary address identifier for coins like XRP,XMR etc.)
amount | 1
recvWindow | 5000
name | addressName (Description of the address)
timestamp | 1508396497000
signature  | 0x56c339a60907073e0f3be8b27d41c79b256e859fb8a268076e5e039fc0398110


### Example 1: As a query string
* **queryString:** asset=ETH&address=0x001866Ae5B3de6cAa5a51543FD9fB64f524F5478&amount=1&recvWindow=5000&name=test&timestamp=1510903211000


### Withdraw
```
POST /wapi/v3/withdraw.html (HMAC SHA256)
```
Submit a withdraw request.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
asset	|  STRING |	YES	
address	 | STRING | YES	
addressTag | STRING | NO | Secondary address identifier for coins like ETH,NXM etc.
amount | DECIMAL | YES	
name | STRING | NO | Description of the address
recvWindow | LONG | NO	
timestamp | LONG | YES	
**Response:**
```javascript
[
{
    "msg": "success",
    "success": true,
    "id":"7553fea8e94b4a5593d507237e5a5bdb"
}
]
```


### Deposit History (USER_DATA)
```
GET /wapi/v3/depositHistory.html (HMAC SHA256)
```
Fetch deposit history.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
asset | STRING | NO	
status | INT | NO | 0(0:pending,1:success)
startTime | LONG | NO	
endTime | LONG | NO	
recvWindow | LONG | NO	
timestamp | LONG | YES	


**Response:**
```javascript
{
    "depositList": [
        {
            "insertTime": 1508198532000,
            "amount": 0.04670582,
            "asset": "ETH",
            "address": "0x001866Ae5B3de6cAa5a51543FD9fB64f524F5478",
            "txId": "0x56c339a60907073e0f3be8b27d41c79b256e859fb8a268076e5e039fc0398110",
            "status": 1
        },
    ],
    "success": true
}
```

### Withdraw History (USER_DATA)
```
GET /wapi/v3/withdrawHistory.html (HMAC SHA256)
```
Fetch withdraw history.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
asset | STRING | NO	
status | INT | NO | 0(0:Email Sent,1:Cancelled 2:Awaiting Approval 3:Rejected 4:Processing 5:Failure 6Completed)
startTime | LONG | NO	
endTime | LONG | NO	
recvWindow | LONG | NO	
timestamp | LONG | YES	


### Deposit Address (USER_DATA)
```
GET  /wapi/v3/depositAddress.html (HMAC SHA256)
```
Fetch deposit address.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
asset | STRING | YES	
status | Boolean |NO
recvWindow | LONG | NO	
timestamp | LONG | YES	

**Response:**
```javascript
[
{
    "address": "0x001866ae5b3de6caa5a51543fd9fb64f524f5478",
    "success": true,
    "addressTag": "3977711",
    "asset": "NXM"
}
]
```

### Account Status (USER_DATA)
```
GET /wapi/v3/accountStatus.html
```
Fetch account status detail.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO	
timestamp | LONG | YES	

**Response:**
```javascript
{
    "msg": "Order failed:Low Order fill rate! Will be reactivated after 5 minutes.",
    "success": true,
    "objs": [
        "5"
    ]
}
```

### System Status (System)
```
GET /wapi/v3/systemStatus.html
```

Fetch system status.
**Response:**
```javascript
{ 
    "status": 0,              // 0: normal，1：system maintenance
    "msg": "normal"           // normal or system maintenance
}
```

### DustLog (USER_DATA)
```
GET /wapi/v3/userAssetDribbletLog.html   (HMAC SHA256)
```
Fetch small amounts of assets exchanged NXM records.


**Weight:**
1


**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO  
timestamp | LONG | YES  

**Response:**
```javascript
{
    "success": true, 
    "results": {
        "total": 2,   //Total counts of exchange
        "rows": [
            {
                "transfered_total": "0.00132256",//Total transfered NXM amount for this exchange.
                "service_charge_total": "0.00002699",   //Total service charge amount for this exchange.
                "tran_id": 4359321,
                "logs": [           //Details of  this exchange.
                    {
                        "tranId": 4359321,
                        "serviceChargeAmount": "0.000009",
                        "uid": "10000015",
                        "amount": "0.0009",
                        "operateTime": "2018-05-03 17:07:04",
                        "transferedAmount": "0.000441",
                        "fromAsset": "USDT"
                    },
                    {
                        "tranId": 4359321,
                        "serviceChargeAmount": "0.00001799",
                        "uid": "10000015",
                        "amount": "0.0009",
                        "operateTime": "2018-05-03 17:07:04",
                        "transferedAmount": "0.00088156",
                        "fromAsset": "ETH"
                    }
                ],
                "operate_time": "2018-05-03 17:07:04" //The time of this exchange.
            },
            {
                "transfered_total": "0.00058795",
                "service_charge_total": "0.000012",
                "tran_id": 4357015,
                "logs": [       // Details of  this exchange.
                    {
                        "tranId": 4357015,
                        "serviceChargeAmount": "0.00001",
                        "uid": "10000015",
                        "amount": "0.001",
                        "operateTime": "2018-05-02 13:52:24",
                        "transferedAmount": "0.00049",
                        "fromAsset": "USDT"
                    },
                    {
                        "tranId": 4357015,
                        "serviceChargeAmount": "0.000002",
                        "uid": "10000015",
                        "amount": "0.0001",
                        "operateTime": "2018-05-02 13:51:11",
                        "transferedAmount": "0.00009795",
                        "fromAsset": "ETH"
                    }
                ],
                "operate_time": "2018-05-02 13:51:11"
            }
        ]
    }
}
```

### Trade Fee (USER_DATA)
```
GET  /wapi/v3/tradeFee.html (HMAC SHA256)
```
Fetch trade fee.


**Weight:**
1


**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO  
timestamp | LONG | YES  
symbol | STRING | NO

**Response:**
```javascript
{
	"tradeFee": [{
		"symbol": "NXM/ETC",
		"maker": 0.9000,
		"taker": 1.0000
	}, {
		"symbol": "ETH/NXM",
		"maker": 0.3000,
		"taker": 0.3000
	}],
	"success": true
}
```


### Asset Detail (USER_DATA)
```
GET  /wapi/v3/assetDetail.html (HMAC SHA256)
```
Fetch asset detail.


**Weight:**
1


**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO  
timestamp | LONG | YES  

**Response:**
```javascript
{
    "success": true,
    "assetDetail": {
        "NXM": {
            "minWithdrawAmount": "70.00000000", //min withdraw amount
            "depositStatus": false,//deposit status
            "withdrawFee": 35, // withdraw fee
            "withdrawStatus": true, //withdraw status
            "depositTip": "Delisted, Deposit Suspended" //reason
        },
        "ETH": {
            "minWithdrawAmount": "0.02000000",
            "depositStatus": true,
            "withdrawFee": 0.01,
            "withdrawStatus": true
        }	
    }
}
```
