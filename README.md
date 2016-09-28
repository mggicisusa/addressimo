FORMAT: 1A

# Addressimo

*Addressimo* is an open source, digital currency address service written in Python.

<img src="http://i.imgur.com/xVbvrZz.png" width="300">

### Download

*Addressimo* is available for download here:
 
[**https://github.com/netkicorp/addressimo**](https://github.com/netkicorp/addressimo)

# Overview

*Addressimo* is an address service. While there are many ways to generate and/or receive an address, the end result always involves
the requesting entity recvieving an address to which they can send value.  The value of a service such as *Addressimo* is that
 it helps to provide additional security and privacy surrounding address generation / requests.
 
Quite simply, a wallet can receive different address types just by making an address request like this:

**https://addressimo_site_url/address/{id}/resolve**
 
## Supported Functionality
 
### Address Generation

* [Static Wallet Addresses](#static-anchor)
* [BIP0032-based HD-Wallet Addresses](#bip32-anchor)

### Address Retrieval

* [BIP0021 / BIP0072 URI](#bituri-anchor)
* [BIP0070 PaymentRequest](#pr-anchor)
* [InvoiceRequests (Encrypted PaymentRequests)](#ir-anchor) 

### Payment Protocol Pathways

* [Full PaymentRequest, Payment, and PaymentACK Processing Support](#fullflow-anchor)
* [PaymentRequest Store & Forward](#sf-anchor)
* [BIP 75](#bip75-anchor) 

<a name="static-anchor"/>

A **Static Address** is a single, non-changing address. Due to the potential for privacy leaks, it is generally not considered best practice to use static addresses,
 but some wallets may only support a single, static address.

<a name="bip32-anchor"/>

**BIP0032** address generation is based on an extended master public key as well as a Redis-stored blockchain used *address
cache* that is maintained via a cronjob. If an address has been used in a transaction output on the blockchain, the next
index will be tried for *[BIP0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)* wallet address generation.

<a name="pr-anchor"/>

**BIP0070 PaymentRequest** generation can use wallet addresses based on a *static wallet address* or a 
*[BIP0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)* wallet address that is generated based 
on the logic previously explained. *[BIP0070 PaymentRequests](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki)* are 
created and signed on demand using the endpoint's configured private key (or configured signer plugin) and x509 cert.

<a name="bituri-anchor"/>

The **Bitcoin URI** is defined in [BIP0021](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki), with extensions
 defined in [BIP0072](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki) in order to support 
 [BIP0070](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) PaymentRequests and later PaymentRequest extensions. 
 The Bitcoin URI was created *"to enable users to easily make payments by simply clicking links on webpages or scanning QR Codes".*
 
| BIP     | Example Bitcoin URI  |
| --- | --- |
| BIP0021 | bitcoin:175tWpb8K1S7NmH4Zx6rewF9WQrcZv245W?amount=50&label=Luke-Jr&message=Donation%20for%20project%20xyz |
| BIP0072 | bitcoin:mq7se9wy2egettFxPbmn99cK8v5AFq55Lx?amount=0.11&r=https://merchant.com/pay.php?h%3D2a8628fc2fbe    |  
 

## Address Resolution
*Addressimo* provides address resolution services based on the configuration of the *Addressimo* endpoint. Depending on that 
configuration, the resolution endpoint (GET) can return:

1. [BIP0072](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki) [Bitcoin URI](#bituri-anchor) in plaintext
2. A PaymentRequest if the URL's bip70 query parameter is set to true

With **BIP75 support**, *Addressimo* supports Store & Forward address resolution services. For BIP75-enabled address 
resolution endpoints, *Addressimo* accepts *InvoiceRequests* in both *ProtocolMessages* and *EncryptedProtocolMessages*

<a name="fullflow-anchor"/>
## Full PaymentRequest, Payment, and PaymentACK Processing
*Addressimo* implements end-to-end [BIP0070](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) PaymentRequest, 
Payment, PaymentACK handling. PaymentRequests are generated via the various methods described in this document. If an 
endpoint is not configured with a payment_url, *Addressimo* will generate a unique payment_url when generating a PaymentRequest 
and assume responsibility for Payment and PaymentACK handling.

Upon receiving a Payment message, *Addressimo* will validate the Payment message according to the specification defined 
for [BIP0070](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki). If a Payment message is deemed valid and 
satisfies all requirements of the generated PaymentRequest, *Addressimo* will submit the Payment to the P2P network and 
respond with a PaymentACK.

**NOTE:** *Addressimo* can only handle Payments for PaymentRequests that are generated by *Addressimo*. *Addressimo* 
does not have the necessary information to handle Payment validation for [Store & Forward](#sf-anchor) or encrypted PaymentRequests.

<a name="sf-anchor"/>
## Store & Forward

Additionally, *Addressimo* can act as a **Store & Forward** server for **pre-signed** PaymentRequests. This is particularly useful for purely
mobile wallets that would otherwise need to be online in order to create and sign *PaymentRequests*. The *Store & Forward* functionality is as follows:

* Register an endpoint
* Add Stored PaymentRequests
* Get Available PaymentRequest Count
* Delete Endpoint

The **Store & Forward PaymentRequests** are available in the same way as the create and sign on demand *PaymentRequests*, 
and are retrieved in the general *Addressimo* address lookup request (described in the top of this document and documented below).

<a name="bip75-anchor"/>
## BIP0075 Support

### Functionality
BIP0075 comprises a set of functionality that allows sending and receiving parties to participate in the Payment Protocol 
while messages are kept encrypted. This allows for the two parties to exchange encrypted InvoiceRequests, PaymentRequests,
 Payments and PaymentACKs without exposing the message details to 3rd parties **or** a *Store & Forward* service such as
 *Addressimo*. This functionality is enabled when the ir_only configuration flag is enabled for an endpoint.
 
Please see [BIP0075](https://github.com/bitcoin/bips/blob/master/bip-0075.mediawiki) for more information
about its updates to the Payment Protocol. Addressimo fully supports BIP0075 as a **Store & Forward** service and
allows both a fully encrypted message exchange as well as the non-encrypted InvoiceRequest submission option.

An example of the fuly encrypted BIP75 Payment Protocol flow can be seen in **functest/functest_ir.py**

### BIP75 Error Communication

As seen in the API documentation below, if an EncryptedPaymentRequest or PaymentACK is not possible, the Receiver can 
respond to the EncryptedPaymentRequest endpoint with an **error_code** and **error_message** for the Receiver. The 
Receiver will then receive the **error_code** and **error_message** the next time they poll for an EncryptedPaymentRequest 
or PaymentACK.

Addressimo **WILL NOT** submit a request to the notification_url with an error.

# Setup

## Requirements
1. Python 2.7.9+
2. RPC access to a running full-node bitcoin server.
3. A running Redis service 

**DB SELECTION NOTE:** Redis is used by default, but you can use any datastore you can write resolver code for. To do this, make sure to 
inherit from BaseResolver and create a PR once complete!

**MEMORY USE NOTE:** A full bitcoin address cache can take 2G+ of space on disk / memory. Please make sure you have enough resources to run the service.


## Installation

##### In order to run *Addressimo* (managed by supervisor), please follow these steps:

    git clone https://github.com/netkicorp/addressimo.git
    pip install supervisor
    mkdir -p /var/log/addressimo
    chown appuser:appuser /var/log/addressimo
    
##### Add this supervisor configuration stanza to */etc/supervisord.conf* after making any changes:
    
    [program:addressimo]
    command = /path/to/bin/python /path/to/bin/gunicorn server:app -b0.0.0.0:5000 -w 4 --timeout 60 --log-level debug --access-logfile /var/log/addressimo/addressimo_access.log --log-file /var/log/addressimo/addressimo.log
    directory = /home/appuser/addressimo
    user = appuser
    numprocs=1
    stdout_logfile=/var/log/addressimo/addressimo.log
    stderr_logfile=/var/log/addressimo/addressimo_err.log
    autostart=true
    autorestart=true
    startsecs=10
    priority=800
    environment= PYTHONPATH="$PYTHONPATH:/use/only/for/virtualenvs"
    
##### Update Addressimo configuration parameters in $ADDRESSIMO_HOME/addressimo/config.py

The configuration parameters are named in order to be understandable. The main things to be concerned with here
are bitcoin service config and redis service config. There are various other parameters that provide defaults for *Addressimo*.
    
##### Add Build Address Cache cronjob:

    PYTHONPATH=/use/only/for/virtualenvs

    * * * * * /path/to/bin/python /home/appuser/addressimo/jobs/build_address_cache.py >> /var/log/addressimo/build_address_cache.log 2>&1
    
**NOTE:** Upon first run, build_address_cache.py will run through the entire blockchain to load the address cache. This **will** 
take some time! In order for *Addressimo* to start resolving endpoints, you will need to wait until the initial load is complete to start-up the service.

Addressimo can act as a pure **Store & Forward** server if config.store_and_forward_only is set to True. This will not required the 
whole bitcoin blockchain to be processed and does not require the Address Cache to be built.

##### Start Addressimo:
    
    supervisor start addressimo    

# Group Addressimo

<a name="auth-anchor"/>
### Endpoint Authentication and Request Validation

Authentication and verification for some endpoints is based on [BitPay's](https://bitpay.com/api) signed request process.
If you're interested in reading through Bitpay's API documentation regarding the X-Identity and X-Signature headers, it 
can be found [here](https://bitpay.com/api#making-requests).

All requests require both the X-Identity and X-Signature headers to be present and valid. The X-Identity header is the hex-encoded
ECDSA public key (DER encoded) for the private key that was used to sign the request. The X-Signature header is the hex-encoded, DER-formatted ECDSA signature
of the message (using SHA256 as the sign digest) that is computed as follows:

privkey.sign( **url** + **request data content**, hashfunc=sha256, sigencode=sigencode_der)

For example, when submitting the following request data to https://addressimo.netki.com/address/longUuid/resolve:

{"key":"value"}

The API request would have a signature that has signed the string: https://addressimo.netki.com/address/longUuid/resolve{"key":"value"}

**NOTE:** An endpoint that requires this method of authentication and verification is denoted with **[REQUIRES AUTHENTICATION](#auth-anchor)**

## Address Resolution [/address/{id}/resolve]

### Immediate Address Resolution [GET]

**NOTE:** The address lookup can return a simple wallet address via the defined JSON format below **OR** it can return a BIP0070 PaymentRequest.
The response type can be determined by looking at the Content-Type in the API response.

+ Parameters

    - id (required, string, `f282ad27e92f4e518a0738dd99469470`) ... Address Resolution ID

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "wallet_address": "1CpLXM15vjULK3ZPGUTDMUcGATGR9xGitv"
        }
        
+ Response 200 (application/bitcoin-paymentrequest)

        SERIALIZED BINARY PROTOBUF PaymentRequest
        
+ Response 400 (application/json)

        {
            "success": false,
            "message": "misconfiguration or bad request message"
        }
        
+ Response 404 (application/json)

        {
            "success": false,
            "message": "not found message"
        }
        
+ Response 405 (application/json)

    **NOTE:** This resolve endpoint requires that an [InvoiceRequest](#ir-anchor)  be created.

    + Headers
    
            Allow: "POST"
            
    + Body
    
            {
                "success": false,
                "message": "Endpoint Requires a valid InvoiceRequest POST to begin the BIP75 Payment Protocol process"
            }
        
+ Response 500 (application/json)

        {
            "success": false,
            "message": error_message
        }
        
### InvoiceRequest Submission [POST]

**[REQUIRES AUTHENTICATION](#auth-anchor)**

In addition to the existing authentication requirement, if an x509 certificate is used in the InvoiceRequest, the request must be 
signed by the x509 certificate's private key. In this case, the process of signing the request would happen two times as defined here:

1. Create InvoiceRequest with the signature field set to ""
2. Sign the InvoiceRequest using the X509 Certificate's Private Key
3. Set the InvoiceRequest's signature field to the signature from Step 2.
4. Sign URL + Serialized InvoiceRequest with the ECDSA Private Key (as described above)

**NOTE:** The Location header returned from the POST call is used to retrieve the pending EncryptedPaymentRequest.

+ Parameters

    - id (required, string, `f282ad27e92f4e518a0738dd99469470`) ... Address Resolution ID

+ Request (application/bitcoin-paymentprotocol-message)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"
            
    + Body

            Serialized ProtocolMessage containing InvoiceRequest
            
+ Request (application/bitcoin-encrypted-paymentprotocol-message)

    + Headers
    
            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"

    + Body
    
            Serialized EncryptedProtocolMessage containing InvoiceRequest
        
+ Response 202

    + Headers
        
            Location: "https://site_url/paymentprotocol/longUuid_payment_id"
    
+ Response 400 (application/json)

        {
            "success": false,
            "message": "Requests including x509 cert must include signature"
        }

+ Response 404 (application/json)

        {
            "success": false,
            "message": "ID Not Recognized"
        }
            

## Store & Forward [/address/{id}/sf]
        
**[REQUIRES AUTHENTICATION](#auth-anchor)**

### Create Store & Forward Endpoint [POST /sf]

The "id" returned in the POST call will need to be used for any further access to the Store & Forward functionality.
   
+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "id': "newly_created_endpoint_id",
            "endpoint": "https://site_url/address/newly_created_endpoint_id/resolve"
        }
        
+ Response 500 (application/json)

        {
            "success": false,
            "message": "error_message"
        }

### Add Presigned PaymentRequests to Store & Forward Endpoint [PUT]

+ Parameters

    + id (required, string, `f282ad27e92f4e518a0738dd99469470`) ... Store & Forward Endpoint ID
   
+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            
    + Body

            {
                "presigned_payment_requests": [
                    "HEX ENCODED PRESIGNED & SERIALIZED PaymentRequest #1",
                    "HEX ENCODED PRESIGNED & SERIALIZED PaymentRequest #2",
                    "HEX ENCODED PRESIGNED & SERIALIZED PaymentRequest #N"
                ]
            }

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "payment_requests_added": 5
        }
        
+ Response 400 (application/json)

        {
            "success": false,
            "message": "bad request error message"
        }
        
+ Response 404 (application/json)

        {
            "success": false,
            "message": "Invalid Identifier"
        }
        
### Delete Store & Forward Endpoint [DELETE]

+ Parameters

    + id (required, string, `f282ad27e92f4e518a0738dd99469470`) ... Store & Forward Endpoint ID
   
+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            
+ Response 204 (application/json)


+ Response 404 (application/json)

        {
            "success": false,
            "message": "Invalid Identifier"
        }
        
### Get Presigned PaymentRequest Count [GET]

+ Parameters

    + id (required, string, `f282ad27e92f4e518a0738dd99469470`) ... Store & Forward Endpoint ID
   
+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "payment_request_count": 5
        }
        
+ Response 404 (application/json)

        {
            "success": false,
            "message": "Invalid Identifier"
        }
        

## Time Endpoint [/time]

### Get Server Time [GET /time]
+ Request (application/json)

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "utime": 1452797108831686
        }

## Payment Protocol Store and Forward Endpoint [/address]

**NOTE:** Some endpoints **[REQUIRE AUTHENTICATION](#auth-anchor)**

This endpoint supports the full BIP 75 implementation. All ProtocolMessage and EncryptedProtocolMessage messages require use of the identifier field to uniquely identify the transaction. Additionally, the signature field is required by *Addressimo*.

### Register Store & Forward Endpoint [POST]

Create a BIP75 Store & Forward Endpoint.

+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "id': "newly_created_endpoint_id",
            "endpoint": "https://site_url/address/newly_created_endpoint_id/resolve"
        }
        
+ Response 500 (application/json)

        {
            "success": false,
            "message": "error_message"
        }

### Retrieve Payment Protocol Messages [GET /address/{id}/paymentprotocol]

Retrieve stored Payment Protocol messages for the given endpoint.

+ Parameters

    - id (required, string, `5935d444b6214296887f4c88e66f5628`) ... Store and Forward Endpoint ID
   
+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "count": 3,
            "protocol_messages": [
                HEX ENCODED, SERIALIZED ProtocolMessage
            ],
            "encrypted_protocol_messages": [
                HEX ENCODED, SERIALIZED EncryptedProtocolMessage,
                HEX ENCODED, SERIALIZED EncryptedProtocolMessage
            ]
        }
        
+ Response 404 (application/json)

        {
            "success": false,
            "message": "Invalid Identifier"
        }
        
+ Response 500 (application/json)

        {
            "success": false,
            "message": "Exception Occurred, Please Try Again Later."
        }

### Submit Payment Protocol Message [POST /address/{id}/paymentprotocol]
Submit Payment Protocol messages (ProtocolMessage / EncryptedProtocolMessage) to a given endpoint. Per BIP75, error communication is handled completely within the Protocol Messages using status_code and status_message to comunicate the error.

+ Parameters

    - id (required, string, `5935d444b6214296887f4c88e66f5628`) ... Store and Forward Endpoint ID

+ Request (application/bitcoin-paymentprotocol-message)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"
    + Body

            Serialized ProtocolMessage

+ Request (application/bitcoin-encrypted-paymentprotocol-message)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"
    + Body

            Serialized EncryptedProtocolMessage

+ Response 200 (application/json)

        {
            "success": true,
            "message": "Payment Protocol message accepted"
        }
               
+ Response 400 (application/json)

        {
            "success": "false",
            "message", "Invalid Request"
        }
        
+ Response 500 (application/json)

        {
            "success": false,
            "message": "error_message"
        }

### Delete Payment Protocol Message [DELETE /address/{id}/paymentprotocol/{identifier}/{messagetype}]

Delete Payment Protocol messages (ProtocolMessage / EncryptedProtocolMessage) from a given endpoint based on transaction identifier and message type.

+ Parameters

    - id (required, string, `5935d444b6214296887f4c88e66f5628`) ... Store and Forward Endpoint ID
    - identifier (required, string, `6d73675f6964656e746966696572`) ... Hex-Encoded Payment Protocol transaction identifier
    - messagetype (required, string, `invoice_request`) ... Any of the values of ProtocolMessageType `(ie., invoice_request, payment_request, payment, payment_ack)`

+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"
    + Body

+ Response 204
        
    
    
## Payment Protocol Transaction Endpoint [/paymentprotocol/{tx_id}]

**NOTE:** Some endpoints **[REQUIRE AUTHENTICATION](#auth-anchor)**

This endpoint supports the full BIP 75 implementation. All ProtocolMessage and EncryptedProtocolMessage messages require use of the identifier field to uniquely identify the transaction. Additionally, the signature field is required by *Addressimo*.

### Retrieve Payment Protocol Messages [GET]

Retrieve stored Payment Protocol messages for the transaction.

+ Parameters

    - tx_id (required, string, `5935d444b6214296887f4c88e66f56285935d444b6214296887f4c88e66f56285935d444b6214296887f4c88e66f5628`) ... Store and Forward BIP75 Transaction ID
   
+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"

+ Response 200 (application/json)

        {
            "success": true,
            "message": "",
            "count": 3,
            "protocol_messages": [
                HEX ENCODED, SERIALIZED ProtocolMessage
            ],
            "encrypted_protocol_messages": [
                HEX ENCODED, SERIALIZED EncryptedProtocolMessage,
                HEX ENCODED, SERIALIZED EncryptedProtocolMessage
            ]
        }
        
+ Response 404 (application/json)

        {
            "success": false,
            "message": "Invalid Identifier"
        }
        
+ Response 500 (application/json)

        {
            "success": false,
            "message": "Exception Occurred, Please Try Again Later."
        }

### Submit Payment Protocol Message [POST]

Submit Payment Protocol messages (ProtocolMessage / EncryptedProtocolMessage) for a given transaction. Per BIP75, error communication is handled completely within the Protocol Messages using status_code and status_message to comunicate the error.

+ Parameters

    - tx_id (required, string, `5935d444b6214296887f4c88e66f56285935d444b6214296887f4c88e66f56285935d444b6214296887f4c88e66f5628`) ... Store and Forward BIP75 Transaction ID

+ Request (application/bitcoin-paymentprotocol-message)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"
    + Body

            Serialized ProtocolMessage

+ Request (application/bitcoin-encrypted-paymentprotocol-message)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"
    + Body

            Serialized EncryptedProtocolMessage

+ Response 200 (application/json)

        {
            "success": true,
            "message": "Payment Protocol message accepted"
        }
               
+ Response 400 (application/json)

        {
            "success": "false",
            "message", "Invalid Request"
        }
        
+ Response 500 (application/json)

        {
            "success": false,
            "message": "error_message"
        }

### Delete Payment Protocol Message [DELETE /paymentprotocol/{tx_id}/{identifier}/{messagetype}]

Delete Payment Protocol messages (ProtocolMessage / EncryptedProtocolMessage) for a given transaction based on transaction identifier and message type.

+ Parameters

    - tx_id (required, string, `5935d444b6214296887f4c88e66f56285935d444b6214296887f4c88e66f56285935d444b6214296887f4c88e66f5628`) ... Store and Forward BIP75 Transaction ID
    - identifier (required, string, `6d73675f6964656e746966696572`) ... Hex-Encoded Payment Protocol transaction identifier
    - messagetype (required, string, `invoice_request`) ... Any of the values of ProtocolMessageType `(ie., invoice_request, payment_request, payment, payment_ack)`

+ Request (application/json)

    + Headers

            X-Identity: "HEX ENCODED ECDSA PUBLIC KEY"
            X-Signature: "HEX ENCODED ECDSA MESSAGE SIGNATURE"
            Content-Transfer-Encoding: "binary"
    + Body

+ Response 204
    