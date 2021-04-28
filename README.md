# API DOCUMENT FOR CIPA SCHEMES CONSOLIDATION

# INTRODUCTION 
This document describes the steps to be taken by merchants that want to receive Scheme (VisaCard, MasterCard, Amex, VerveCard and PayAttitude) details on their respective sites and process the transactions on our Payment Gateway. 

**NOTE**: This document is for merchants that have been setup (by E-Commerce Ops) on the payment gateway.


## INTEGRATION PROCEDURE

* GO TO https://test.payarena.com/prospectivemerchants on your browser
* Fill in details in the form that will be presented to you.
* Submit the form
* Test Configuration details will be forwarded to you via the email you provided. This will include the following: 
   Merchant ID
   Cryptographic Key
* The Merchant ID and Cryptographic key are used to identify a Webshop on UP MPI each time a Webshop sends a payment request, i.e. a create order request to UP MPI.
* Review this documentation and commence implementation
* Review UAT documentation and ensure that all UAT requirements are met on merchant site
* System Integration Test and User Acceptance Test with Unified Payments team
* On successful UAT, Unified Payments team will initiate Go-live process and provide production parameters
* Apply production parameters and confirm successful pilot



## IMPLEMENTATION
For a seamless integration, kindly follow the outlined steps below:
**Step 1:** Create a json representation of data elements as follows: 
```json
{
  "id": "MERCHANTID",
	"description": "payment for goods",
	"amount": 200, 
	"fee": 0,
	 "currency": "566",
	"returnUrl": "http://mywebsite.com/returnurl",
	"secretKey": "AJAHD45S45F4S4S45AS45D54S",
	"scheme": "",
	 "vendorId": "",
	 "parameter": "",
	"count" : 0
	 }
   ```
   
**Step 2:** Send the json data created above as a POST request to https://test.payarena.com/Aggregator, the header should specify Accept and Content-Type as application/json. This will return an id for the transaction posted above.

**Step 3:** Generate a SHA1 string of your secret key.

**Step 4:** Create a json representation of the card details as follows:
```json
{
  "secretKey": "AJAHD45S45F4S4S45AS45D54S",
  "scheme": "visa",
  "cardNumber": "4999082100029373",
  "expiry": "01/19",
  "cvv": "126",
  "cardholder": "",
  "mobile": "",
  "pin": ""
}
 ```
 
**NOTE: **
When the scheme is either Visa or Mastercard or Amex, fields- cardholder, mobile and pin should be empty.
When scheme is Verve, all fields must be supplied but when scheme is PayAttitude, only the secret key and mobile fields should be populated, all others should be empty as seen below.

```json
{
  "secretKey": "AJAHD45S45F4S4S45AS45D54S",
  "scheme": "",
  "cardNumber": "",
  "expiry": "",
  "cvv": "",
  "cardholder": "",
  "mobile": "07087419908",
  "pin": ""
}
 ```


When it is recurring payment use the json representation details as seen below.

 ```json
{
"secretKey": "AJAHD45S45F4S4S45AS45D54S",
"scheme": "visa",
"cardNumber": "4999082100029373",
"expiry": "01/19",
"cvv": "126",
"cardholder": "",
"mobile": "",
"pin": "",
"endRecurr":"",
"frequency":"", "isRecurring":false, "OrderExpirationPeriod":0
}
```
**NOTE:**
For recurring payments, the frequency should contain the number of times the payment can be done in a given period,, the isRecurring field should be set to true, the endRecurr should contain the end date for the recurring payment( in the format yyyyMMdd) and the OrderExpirationPeriod contain the interval, in days, in which the payment will reoccur.

**Step 5:** Using the AES algorithm, sample code provided below, encrypt the json data in Step 4 above using the first 16 characters of the string obtained in Step 3. See an example below:

 **C#** 
```
public static string Encrypt(byte[] dataToEncrypt, byte[] key, byte[] iv)
{
	using (var aes = new AesCryptoServiceProvider())
	 {
		aes.Mode = CipherMode.CBC;
	 	aes.Padding = PaddingMode.PKCS7;
		aes.Key = key;
	   	aes.IV = iv;
		using (var memoryStream = new MemoryStream())
		 {
		var cryptoStream = new CryptoStream(memoryStream, aes.CreateEncryptor(), CryptoStreamMode.Write);
	 cryptoStream.Write(dataToEncrypt, 0, dataToEncrypt.Length); cryptoStream.FlushFinalBlock();
	 return memoryStream.ToArray().Aggregate("", (current, txt) => current + txt.ToString("X2");
}

}

} 
```

where dataToEncrypt is a byte array of the json card details in step 4 above and key and iv are byte arrays of the first 16 characters in step 3 above.

**Step 6:** 

## Card
Redirect the customer to the endpoint below as a GET request to complete the transaction: https://test.payarena.com/Home/TransactionPost/TRANSACTIONID?mid=MERCHANTID&payload=ENCRYPTEDDATA  
where:
* Transactionid was gotten in step 2 above, 
* Merchantid is unique ID used to identify the merchant an 
* Encrypteddata gotten in step 5 above.

After the transaction is completed, the Payment Gateway will postback the below data:
```json
{ "trxId": "1234567", "approved": "true", "status": "APPROVED"}
```

## PayAttitude: 

For payattitude transactions a POST to the below URL(endpoint) with the payload is required to complete the transaction: 
URL
https://test.payarena.com/Home/PayAttitudeTransactionPost
Request Payload
```json
{
  "Id": "TRANSACTIONID",
  "Mid": "MERCHANTID",
  "Payload": "ENCRYPTEDDATA"
}
```
Set Accept to application/json in the request header.

**Response**
```json
{
    "Order Id": "TRANSACTION ID",
    "Amount": "TRANSACTION AMOUNT",
    "Description": "DESCRIPTION OF TRANSACTION",
    "Currency": "TRANSACTION CURRENCY",
    "Status": "TRANSACTION STATUS",
    "PAN": "CUSTOMER PHONE NUMBER",
    "TranTime": "TRANSACTION DATETIME",
    "StatusDescription": "DESCRIPTION OF THE TRANSACTION STATUS"
}
```

**Note:** Upon receipt of the payment request, the PayAttitude transaction takes about 90seconds to be completed. If the customer does not authenticate the transaction within that time, a timeout response will be sent back to the requestor.
 
**TRANSACTION STATUS QUERY** 

To query the status of a transaction, send a GET request in the format below. 
https://test.payarena.com/Status/Transactionid . Where transactionID is the ID received as a response after the order was created.
The status is the status of the transaction, this could be either of the following in the table below:

|Status | Description |
| --- | --- |
| Approved/Approved Successful | A transaction is approved when the customer is successfully debited and value for transaction is received. |
| Cancelled| A transaction is cancelled when the customer decides to not enter payment details and returns back to the merchant site. |
| Declined|A transaction is declined when one of the following happens:Unsucccessful authentication,Unsuccessful authorization,System error |
| Declined| A transaction is declined when one of the following happens:Unsucccessful authentication,Unsuccessful authorization,System error |
| Initiated| A transaction is initiated when the customer abandons a transaction. |

 

**RECURRING TRANSACTION STATUS QUERY**

For recurring payment, a SessionId field is returned when a getstatus request is made (this is valid only for an approved transaction) . 
The SessionId is a compulsory field when subsequent payments are to be made.

After the first successful transaction and a Status query has returned the SessionId among other parameters, follow steps below to make a recurring payment:

**Step 1:** Create a json representation of the data elements below
```json
{
"SecretKey": "2BC80A5EB5BB6A64A772F9806A7E9A0B16702043AB475DC4",
"Scheme": "visa", 
"Amount": 500,
"Fee": 0,
"SessionId": "DE4B683DBCA6D3BD81041D148BCEDF46",
"Currency": "566",
"CustomerEmail": "text@text.com", 
"CustomerName": "Unified Payments", 
"Description": "EKEDC",
"ReturnUrl": "http://localhost:7501/transaction/pgpostback"
};
```
**Step 2: ** Encrypt the json data above using AES Algorithm specified earlier. 
**Step 3:** Initiate a HTTP GET request to the url below with the encrypted string in Step 2 above. 
https://test.payarena.com/Home/RecurringTransaction/MERCHANTID?payload=ENCRYPTEDDA TA. A json object containing the status of the transaction will be returned after the transaction has been processed.





## TRANSACTION PROCESS FLOW 

![TRANSACTION PROCESS FLOW](/img/process.png)

## Transaction Process Flow (Explained)
* Customer checks out from WebShop and enters payment details
* The aggregator captures the data and sends it to UP MPI (in the case of card data its encrypts the data before sending to the UP MPI)
* UP MPI collects card details and sends authentication requests to the 3DSecure Provider.
* The 3D Secure Provider contacts the issuer/cardholder to authenticate the transaction
* The issuer/cardholder authenticates transaction and sends a response to 3D Secure Provider
* The 3D Secure Provider sends the authentication response to UP MP
* UP MPI sends an authorization request to the Switch 
* UP Switch sends the authorization request to Scheme
* The Scheme contacts issuer to authorize  
* The Issuer authorizes the the transaction and sends the authorization response to the scheme 
* The Scheme sends the authorization response to the Switch
* The Switch sends the authorization response to UP MPI.
* UP MPI sends the payment confirmation response to the aggregator.
* The aggregator  sends the payment confirmation response to the webshope. The Webshop logs the response and displays payment confirmation to the 
customer with the details received from UP MPI.







## Contact
Created by [@eplatform(eplatform@up-ng.com) - feel free to contact us for clerifications.
