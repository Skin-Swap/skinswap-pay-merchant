
# Skinswap Pay Merchant API docs ❤

Pay.skinswap.com's documentation for handling the API.




## API Reference
Main Route https://pay.skinswap.com/

Information about **IPN (Instant Payment Notification)** can be found in the FAQ

#### ⚠ IMPORTANT ⚠
All API calls requires a signature and merchant ID to be sent along.

*Merchant ID*
* /api/merchant/**:merchantId**/random

*Signature*

* The signature is included in the body of each API call.
* The signature is a hash consisting of your API key and the data you are sending/receiving
* Generation details can be found further down

**Conclusion** ✔

| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `merchantId` | `string` | **Required** Your Merchant Id |
| `signature` | `string` | **Required** Signature of body|

#### DEPOSIT URL

```https
  GET /api/merchant/:merchantId/deposit_url
```

| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `steamid` | `string` | **Required**. The users Steamid64 |
| `tradeurl` | `string` | **Required**. The users tradeurl |
| `appids` | `string` | **Required**. APPID. Either 252490 - 730 - 730:252490 |

#### GET BALANCE OF MERCHANT ACCOUNT

```https
  GET /api/merchant/:merchantId/balance
```

| Parameter | Type     | Description                       |
| :-------- | :------- | :-------------------------------- |
Only signature of empty body needs to be sent

#### CHECK ORDER ID

```https
  GET /api/merchant/:merchantId/order_id
```

| Parameter | Type     | Description                       |
| :-------- | :------- | :-------------------------------- |
| `order_id` | `number` | **Required** Order id|

## Generating a Signature 👀🔓
```javascript
const crypto = require("crypto");
const secret = "YOUR MERCHANT API KEY"

const sha256 = (string) => {
    return String(crypto.createHash('sha256').update(string).digest('hex'));
}

const buildSignature = (data, secret) =>
```javascript
const secret = "YOUR API KEY"

const sha256 = (string) => {
    return String(crypto.createHash('sha256').update(string).digest('hex'));
}

const buildSignature = (data, secret) => {
  let signatureString = "";

  Object.keys(data).sort().forEach((key) => {
      if (key === "signature") return;
      if (typeof data[key] === "object") return;
      signatureString += data[key];
  })
  return sha256(`${signatureString}${secret}`);
}

```


## FAQ 😎

#### What is generally returned?

The server will respond with an object.
```javascript

// GET:order_id example response 
  {
    "data": {
        "id": ,
        "steamid": "",
        "tradeurl": "",
        "merchant_id": "",
        "status": "",
        "timestamp": "2000-00-00T00:00:00.000Z",
        "signature": ""
    },
    "error": false //Or true
}

```

If an error occured. Error will be set to true.

#### IPN? 

The server will ONLY send an IPN call IF the user's trade has been successfull/gone through *(steam state 3)*
The server will send a maximum of 3 IPN calls, each with a minutes wait between each call.

Your server needs to respond with 

```javascript

// What pay.skinswap.com want's to receive after sending IPN calls.
  {
    "data": {
        "success": true
    }
}

```
## ⚠  Warning 📣
Make sure to only credit the user ONCE. The IPN call might be sent more than one time.
## Support

For support please contact **Skinswap** on Discord. Costs may apply 💸




## Authors

- [@Broder](401)
- [@Rey](401)


❤ Made with Love by your Broder ❤