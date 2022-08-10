
# Skinswap Pay Merchant API docs

## Before you start

### Payment flow

---

You request an url from Skinswap and direct your user to that url. After a successful trade, our system will call your backend (IPN) to notify you. 

### Merchant ID 

---

We use Merchant ID to identify your account in our system. You should have received this from Skinswap representative.

This is required for many of the API calls.

Example of Merchant ID (non-functioning, can't be used for testing) ` 1abf23dac130 `

### API key

---

API key is used to create signatures, that secure the communications between Skinswap and your system. You should have received this from Skinswap representative.

This is required for many of the API calls.

Example of API key (non-functioning, can't be used for testing) ` 9f8cf8b3-9965-5720-ad1b-f2f4feffdd6f `


### Signatures

---

#### Creating a signature

---

NodeJS example code. Later examples assumes same directory and filename `createSignature.js`

```javascript
const crypto = require("crypto")

// Generates a SHA256 hash
function sha256(string){
  return String(crypto.createHash("sha256").update(string).digest("hex"));
}

function createSignature(data, skinswapApiKey){
  /**
   * data is what you're sending to Skinswap api
   * 
   * This doesn't sign the data you're sending, it returns the signature string or false
   */
    if(!requestData || typeof requestData !== "object"){
        console.error(`Error verifying signature, parameter "requestData" is not an object (was called with "${requestData}") `);
        return false;
    }
    
    if(!skinswapApiKey){
        console.error(`Error verifying signature, missing parameter "skinswapApiKey" (was called with "${skinswapApiKey}") `);
        return false;
    }
    let signatureString = "";
    
    Object.keys(data).sort().forEach((key) => {
    if (key === "signature") return;
    if (typeof data[key] === "object") return;
    signatureString += data[key];
    });
    
    return sha256(`${signatureString}${skinswapApiKey}`); // Returns the signature
}

module.exports = createSignature;
```

#### Signing a request data

---

NodeJS example code. Later examples assumes same directory and filename `signRequestData.js`

```javascript
const createSignature = require("./createSignature.js")

function signRequestData(requestData, skinswapApiKey){
  if(typeof requestData !== "object"){
    console.error(`Error verifying signature, parameter "requestData" is not an object (was called with "${requestData}") `);
    return false;
  }
  if(!skinswapApiKey){
    console.error(`Error verifying signature, missing parameter "skinswapApiKey" (was called with "${skinswapApiKey}") `);
    return false;
  }

  const signature = createSignature(requestData, skinswapApiKey);
  return Object.assign({}, requestData, {signature});
}

module.exports = signRequestData;
```

#### Verifying Response data (and IPN request data) signature

---

NodeJS example code. Later examples assumes same directory and filename `verifyDataSignature.js`

```javascript
const createSignature = require("./createSignature.js")

function verifyDataSignature(requestData, skinswapApiKey){
  if(!requestData || typeof requestData !== "object"){
    console.error(`Error verifying signature, parameter "requestData" is not an object (was called with "${requestData}") `);
    return false;
  }
  if(typeof requestData.error !== "undefined" && typeof requestData.data === "object"){
    console.error(`Error verifying signature, parameter "requestData" looks like it's the whole response from pay.skinswap.com (it's an object containing a property called "error" and it has a property called "data", that is an object`);
    return false;
  }
  
  if(!skinswapApiKey){
    console.error(`Error verifying signature, missing parameter "skinswapApiKey" (was called with "${skinswapApiKey}") `);
    return false;
  }
  
  const signature = requestData.signature;
  const expectedSignature = createSignature(requestData);
  
  if(signature !== expectedSignature){
    console.error(`verifyDataSignature() failed, provided signature: "${signature}", computed signature: "${expectedSignature}"`);

    return false;
  }
  return true;
}

module.exports = verifyDataSignature;
```
#### Example implementation of using signatures

---

Example implementation of an `async` function, that fetches the Skinswap deposit url.

It performs a`GET` request to  `/api/merchant/<merchantId>/deposit_url` and validates the response signature

#### Parameters

| Parameter                | Type     | Required   | Description                                    |
|:-------------------------|:---------|:-----------|:-----------------------------------------------|
| `requestParams`          | `object` | yes        | Object containing required details             |
| `requestParams.steamid`  | `string` | yes        | The users Steamid64                            |
| `requestParams.tradeurl` | `string` | yes        | The users tradeurl                             |
| `requestParams.appids`   | `string` | yes        | Accepted values: "252490", "730", "730:252490" |
| `skinswapApiKey`         | `string` | yes        |  |
| `skinswapMerchantId`     | `string` | yes        |  |

#### Returns

Example of return on success:

```
   {  
     data:{
       deposit_url: "https://pay.skinswap.com/deposit/token=eyJhbG....",
       order_id: 4,
       creation_time: 1655670816436,
       signature: 'b295806c339522f3b68ab49c5a521b83dac324af26e23926c671f6cdffee352e'
     },
     error: false
  }
```
Example of return on failure:

```
   {  
     error: true,
     msg: "Signature verification failed"
  }
```

#### Example async function implementation

NodeJS example code. Later examples assumes same directory and filename `getDepositUrl.js`

```javascript

const axios               = require("axios");
const signRequestData     = require("./signRequestData.js");
const verifyDataSignature = require("./verifyDataSignature.js");

/**
 *
 * @param {Object} requestParams
 * @param {String} requestParams.steamid Steam64
 * @param {String} requestParams.tradeurl
 * @param {String} requestParams.appids
 * @param {String} skinswapApiKey
 * @param {String} skinswapMerchantId
 * @returns {{msg: string, error: boolean}}
 */
function validateRequest(requestParams, skinswapApiKey, skinswapMerchantId){
  
  if(typeof requestParams !== "object"){
    console.error(`getDepositUrl() API call to Skinswap failed, missing or invalid parameter "requestParams", expecting Object, called with: "${requestParams}"`);

    return {
      error: true,
      msg: "Missing or invalid parameter 'requestParams'"
    };
  }
  
  if(typeof requestParams.steamid !== "string"){
    console.error(`getDepositUrl() API call to Skinswap failed, missing or invalid parameter "requestParams.steamid", expecting string, called with: "${requestParams.steamid}"`);

    return {
      error: true,
      msg: "Missing or invalid parameter 'requestParams.steamid'"
    };
  }
  
  if(typeof requestParams.tradeurl !== "string"){
    console.error(`getDepositUrl() API call to Skinswap failed, missing or invalid parameter "requestParams.tradeurl", expecting string, called with: "${requestParams.tradeurl}"`);

    return {
      error: true,
      msg: "Missing or invalid parameter 'requestParams.tradeurl'"
    };
  }
  if(typeof requestParams.appids !== "string"){
    console.error(`getDepositUrl() API call to Skinswap failed, missing or invalid parameter "requestParams.appids", expecting string, called with: "${requestParams.appids}"`);

    return {
      error: true,
      msg: "Missing or invalid parameter 'requestParams.appids'"
    };
  }
  if(!(requestParams.appids !== "730" || requestParams.appids !== "252490" || 
    requestParams.appids !== "730:252490")){
    
    console.error(`getDepositUrl() API call to Skinswap nvalid parameter "requestParams.appids", accepted values: "730", "252490", "730:252490", called with: "${requestParams.appids}"`);

    return {
      error: true,
      msg: "Invalid value in parameter 'requestParams.appids'"
    };
  }

  if(typeof skinswapApiKey !== "string"){
    console.error(`Missing or invalid parameter "skinswapApiKey" (was called with "${skinswapApiKey}") `);
    return {
      error: true,
      msg: "Missing or invalid parameter 'skinswapApiKey'"
    };
  }
  
  if(typeof skinswapMerchantId !== "string"){
    console.error(`Missing or invalid parameter "skinswapMerchantId" (was called with "${skinswapMerchantId}") `);
    return {
      error: true,
      msg: "Missing or invalid parameter 'skinswapMerchantId'"
    };
  }
  
  return {error: false, msg: ""};
}

/**
 * 
 * @param {Object} requestParams
 * @param {String} requestParams.steamid Steam64
 * @param {String} requestParams.tradeurl 
 * @param {String} requestParams.appids 
 * @param {String} skinswapApiKey
 * @param {String} skinswapMerchantId
 * 
 * @returns {Promise<{msg: string, axiosResponse: AxiosResponse<any>, error: boolean}|any|{data}|T|{msg: string, error: boolean}|{msg: string, error: boolean}|{msg: string, error: boolean}|{msg: string, error: boolean}>}
 */
async function getDepositUrl(requestParams, skinswapApiKey, skinswapMerchantId){
  
  const requestValidationResult = validateRequest(requestParams, skinswapApiKey, skinswapMerchantId);
  
  if(requestValidationResult.error){
    return requestValidationResult;
  }
  
  const requestData = signRequestData({
    steamid   : requestParams.steamid,
    tradeurl  : requestParams.tradeurl,
    appids    : requestParams.appids
  }, skinswapApiKey);

  console.info( `Making an API call to Skinswap, steamId: '${requestData.steamid}', tradeUrl: '${requestData.tradeurl}', appids: '${requestData.appids}'`);
  
  const merchantId              = skinswapMerchantId;

  const skinswapApiUrl          = "https://pay.skinswap.com/api";
  const skinswapRequestBaseUrl  = `/merchant/${merchantId}/deposit_url`;
  const skinwapRequestUrl       = `${skinswapApiUrl}${skinswapRequestBaseUrl}`;

  try{
    const response = await axios({
      method  : "GET",
      url     : skinwapRequestUrl,
      data    : requestParams,
    });
    
    /**
     * Expected response example:
     *  {  
     *    data:{
     *      deposit_url: "https://pay.skinswap.com/deposit/token=eyJhbG....",
     *      order_id: 4,
     *      creation_time: 1655670816436,
     *      signature: 'b295806c339522f3b68ab49c5a521b83dac324af26e23926c671f6cdffee352e'
     *    },
     *    error: false
     * }
     */
    
    const skinswapResponseData = response.data;
    if(skinswapResponseData.error || !skinswapResponseData.data){
      console.error(`API call to Skinswap failed, error message: "${skinswapResponseData.msg}"`);
      
      return skinswapResponseData;
    }
    console.info(``);
    
    if(!verifyDataSignature(skinswapResponseData.data,skinswapApiKey)){
      /**
       * This really shouldn't happen. It means either Skinswap 
       * is creating incorrect signatures or someone else is.
       */
      console.error(`API call to Skinswap failed, error message: "${skinswapResponseData.msg}"`);
      
      return {
        error         : true, 
        msg           : "Signature verification failed",
        axiosResponse : response
      };
    }
    
    return skinswapResponseData;

  }catch (error){
    const skinswapResponseData = error.response.data;
    console.error(`request to '${skinwapRequestUrl}' FAILED, message: "${skinswapResponseData.msg}"`);
    skinswapResponseData.AxiosError = error;
    return  skinswapResponseData;
  }
}

module.exports = getDepositUrl;

```


### IPN (webhook notifying successful trades)

---

klkjlkj

## API Reference
Main Route `https://pay.skinswap.com/`

#### DEPOSIT URL

---

This endpoint returns an URL that your user needs to be redirected.

- Method: `GET`
- Url: `/api/merchant/<merchantId>/deposit_url`

`**Note: replace `<merchantId> with your actual merchant id**`

#### Request parameters

| Parameter   | Type     | Required   | Description                                    |
|:------------| :------- |:-----------|:-----------------------------------------------|
| `steamid`   | `string` | yes        | The users Steamid64                            |
| `tradeurl`  | `string` | yes        | The users Steam tradeurl                       |
| `appids`    | `string` | yes        | Accepted values: "252490", "730", "730:252490" |
| `signature` | `string` | yes        | Hash of the parameters                         |

#### Example response, successful call

- HTTP status code `200`
- Content type `json`

Response body
```json
{
    "data": {
        "deposit_url": "https://pay.skinswap.com/deposit/token=eyJhbG....9580",
        "order_id": 4,
        "creation_time": 1655670816436,
        "signature": "b295806c339522f3b68ab49c5a521b83dac324af26e23926c671f6cdffee352e"
    },
    "error": false
}
```
#### Example response, failed call
- HTTP status code `400`
- Content type `json`
  
Response body
```json
{
    "error": true,
    "msg": "No merchantID provided!"
}
```

#### GET BALANCE OF MERCHANT ACCOUNT

---

- Method: `GET`
- Url: `/api/merchant/<merchantId>/balance`

**Note: replace `<merchantId>` with your actual merchant id**`


#### Request parameters
| Parameter   | Type     | Required   | Description                                    |
|:------------| :------- |:-----------|:-----------------------------------------------|
| `signature` | `string` | yes        | Hash of the parameters                         |
    Only signature of empty body needs to be sent

#### Example response, successful call

- HTTP status code `200`
- Content type `json`

Response body
```json
{
    "data": {
       
      "signature": "b295806c339522f3b68ab49c5a521b83dac324af26e23926c671f6cdffee352e"
    },
    "error": false
}
```
#### Example response, failed call
- HTTP status code `400`
- Content type `json`

Response body
```json
{
    "error": true,
    "msg": "No merchantID provided!"
}
```

#### CHECK ORDER ID

---

- Method: `GET`
- Url: `/api/merchant/<merchantId>/order_id`

**Note: replace `<merchantId>` with your actual merchant id**`

#### Request parameters

| Parameter   | Type     | Required   | Description            |
|:------------| :------- |:-----------|:-----------------------|
| `order_id` | `number` | yes        | Order id               |
| `signature` | `string` | yes        | Hash of the parameters |
#### Example response, successful call

- HTTP status code `200`
- Content type `json`

Response body
```json
{
    "data": {
        "order_id": 4,
        "creation_time": 1655670816436,
        "signature": "b295806c339522f3b68ab49c5a521b83dac324af26e23926c671f6cdffee352e"
    },
    "error": false
}
```
#### Example response, failed call
- HTTP status code `400`
- Content type `json`

Response body
```json
{
    "error": true,
    "msg": "No merchantID provided!"
}
```



## IPN (webhooks)

---

The server will ONLY send an IPN call IF the user's trade has been successfull/gone through *(steam state 3)*
The server will send a maximum of 3 IPN calls, each with a minutes wait between each call.

### Expected response

- Status code: `200`
- Content type: `json`

Response body

```json
{
   "data": {
     "success": true
  }
}
```

## ⚠  Warning 📣

---
### Make sure to only credit the user ONCE. The IPN call might be sent more than one time.


## FAQ 😎

---

## Support

---

For support please contact **Skinswap** on Discord. Costs may apply 💸

## Authors

---

- [@Broder](401)
- [@Rey](401)


❤ Made with Love by your Broder ❤
