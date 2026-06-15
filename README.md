# Tella Nearby Sharing Protocol

Protocol version: 2

Nearby Sharing lets you securely share files, fully offline, across platforms and devices, assuring secure, anonymous, encrypted file transfers.

This repository describes the peer-to-peer file sharing protocol implemented by all [Tella](https://tella-app.org/) apps. 


## Platform and availability
Nearby Sharing will be available for [Tella Android](https://github.com/Horizontal-org/Tella-Android), [Tella iOS](https://github.com/Horizontal-org/Tella-iOS) and [Tella Desktop](https://github.com/Horizontal-org/Tella-Desktop), but it's still under development.

The feature is still in alpha, and it's currently being audited by an independent security firm. It will be launched to production only after the priority security fixes are implemented.

User facing documentation about the feature is available here: https://beta.tella-app.org/nearby-sharing

## Credits
This protocol (and Nearby Sharing feature in Tella in general) is inspired by the [LocalSend project](https://github.com/localsend/localsend), and it uses the local network Wi-Fi without needing an internet connection. 


## Context of use and key features
Nearby Sharing in Tella was designed for contexts of repression and surveillance, including for being able to share sensitive information before, during and after internet shutdowns. Here are some key details:

- Independent of internet: Transfers work with or without an internet connection, even on surveilled or insecure Wi-Fi networks, by establishing a direct connection between devices instead of routing through the internet.
- Works with Personal Hotspots: even if you don't have data on your phone's plan, you can still create a Personal Hotspot, invite the other person to connect to it, and be able to use Nearby Sharing.
- Available on iOS, Android and Computer: there isn't any restrictions on which model of phone, brand or operative system you use. Nearby Sharing is designed to be accessible to any device able to install Tella
- Encrypted: Files move directly from one Tella vault to another, encrypted and secure.
- Anonymous: There’s no concept of “registered users” in Tella. Nearby Sharing connections happen locally, with no trace of who you shared with, where, or when.


## 1- Security Features

* All connections are secured with HTTPS using self-signed certificates generated per device.
* Authentication is mandatory via PIN and IP address, provided through QR code scanning or manual entry.
* Certificates are verified to prevent machine-in-the-middle (MITM) attacks.
* All connections use a specific port: 53320

The TLS versions in use are TLS1.2 and TLS1.3. Implementations pick the highest version
supported by both sender and receiver.


Self-signed TLS certificates are generated and used on both sides of the connection to
establish a mutual TLS (mTLS) connection: the sender verifies the receiver, and
the receiver verifies the sender.

Certificates are generated and used per session, being discarded when a session ends.

## 2- Connection Authentication

All connections require authentication, either via QR code or manually.

As part of authentication, the receiver assigns a session ID and returns it to
the sender. The session ID lets the receiver know that requests are authorized.
A given session ID is tied to a particular transfer session. The session ID
should be forgotten once the transfer concludes, whether it ends orderly or due
to an error state.

Requests for unknown or concluded transfer sessions should be rejected.

### 2.1- QR authentication (primary method)

#### 2.1.1 Receiver QR code

The receiver device displays a QR code containing:

* Receiver's local IP address
* Port
* Hash of the receiver's TLS certificate
* Connection PIN
* Protocol version number
* Whether sender in step 2 of section *New flow* should directly show hash verification

QR payload:

```json5
{
  ip_address: [String, ..., ..., String],
  port: Number,
  certificate_hash: String,
  pin: String,
  protocol_version: Number,
  sender_show_hash: Boolean
}
```

If for example the receiver is a desktop client, `sender_show_hash` should be set to `true` and
sender in step 2 in section *New flow* should skip display of the QR code and directly show the
hash verification screen.

For the protocol described in the current document, the protocol version should be set to
`protocol_version: 2`

**Note:** `ip_address` is a list of strings, as the receiver may have many different local IP
addresses.

#### 2.1.2 Sender QR code

The sender device displays a QR code containing:

* Hash of the sender's TLS certificate

QR payload:

```json5
{
  certificate_hash: String
}
```

### 2.2- Manual authentication (Fallback Method)

When QR code scanning is not available, the receiver device will display:

* IP address
* 6 digit PIN
* Port number 

After entering the connection information, both the sender and the receiver will display a verification screen. 

### 2.3- Verification screen

Both the sender and the receiver will display a verification screen containing an alphanumeric sequence that encodes the hash of the receiver's TLS certificate.

Both parties will verify that the same sequence is shown on each device before proceeding.

The verification screen will provide two options:
* Confirm and Connect — Proceed with registration if the hashes match.
* Discard and Start Over — Terminate the connection and the user should be returned to the main connection screen.

Example of alphanumeric sequence (SHA-256 hash):

```markdown
87fd 5869 a6b3 e414 
112c 1934 ca00 be77 
b8e4 584c 829a 4536 
490b da9a 3928 be4a
```

**Security Note:** A hash mismatch indicates a potential machine-in-the-middle (MITM) attack.
Users should verify that they are connecting to the intended device and ensure the network environment is secure before retrying.

## 3- Connection Establishment

### 3.1 Initial Ping

`POST /api/v2/ping`

This endpoint initiates a secure handshake between two devices during the manual connection process. It must be called before the register endpoint. Once called, both the sender and receiver display the verification screen.

Response payload

```markdown
{
  "senderShowHash": boolean
}
```

Errors:

|HTTP code|Message|
|--|--|
|429|Too many requests|
|406|Unsupported version|

**Security note**: Sender should only save the data (senderShowHash) from the
ping response after Receiver Hash Verification has been completed. Only after
verifying the Receiver Certificate Hash does sender know they can trust the
ping response payload.

### 3.2- Initial Registration

For QR code authentication, registration is performed immediately after the QR code has been scanned.

For manual authentication, registration is performed after the ping request and once the sender has verified the receiver certificate hash.

The sender should only generate one certificate and use that certificate for configuring the sender's TLS client.

**Note**: Sender needs to attach its certificate information to the registration request and then to all subsequent requests.

After successful registration, mTLS has been established. After registration, requests and
responses that do not have attached certificate information should be automatically rejected.
Also if a computed certificate hash does not match the pinned hash, the corresponding request or
response should also be rejected.

`POST /api/v2/register`

Request payload

```markdown 
{
  pin: "123456",
  nonce: "random-uuid-number",
}
```


Response payload

```markdown
{
  "sessionId": "uuid-session-identifier"
}
```

**Note:** A maximum of 3 invalid requests are allowed.

Errors:

|HTTP code|Message|
|--|--|
|400|Invalid request format|
|401|Invalid PIN|
|403|Rejected| 
|406|Unsupported version|
|409|Active session already exists|
|429|Too many requests|
|500|Server error|

#### New flow

For this flow we use the notation Device A (sender) and Device B (receiver).

**Step 1: Pin receiver certificate**

When sender can scan receiver QR codes: 

* Device B (receiver) presents QR code containing information outlined in *2.1.1 Receiver QR code*
* Device A (sender) scans QR code, parses information, and pins the embedded Receiver Certificate Hash

When sender cannot scan receiver QR codes (broken screen/camera or comms with desktop):

* Device B (receiver) displays manual connection info outlined in *2.2- Manual authentication (Fallback Method)*
* Device A (sender) types in manual connection info and sends ping request outlined in *3.1 Initial Ping*
* Device B (receiver) sends response to ping request
* Device B (receiver) displays the hash of their TLS certificate
* Device A (sender) extracts the receiver's certificate information from the connection info for the ping
  response, hashes it, and displays the Receiver Certificate Hash
* Device A (sender) and Device B (receiver) visually compare the hashes as outlined in *2.3- Verification screen*
* If verification succeds, Device A (sender) pins the Receiver Certificate Hash
* Proceed to step 2

After Device A (sender) has pinned Receiver Certificate Hash, Device A (sender)
will compute each certificate hash on all future responses. For each response
received after pinning, Device A (sender) hashes the certificate information
from the connection and checks the computed hash against the pinned Receiver
Certificate Hash.

**Step 2: Pin sender certificate**

When receiver can scan sender QR codes: 

* Device A (sender) presents QR code containing the sender's certificate hash outlined in *2.1.2 Sender QR code*
* Device B (receiver) scans QR code, parses information, and pins the embedded Sender Certificate Hash
* Device A (sender) sends register request payload outlined in *3.2- Initial Registration*
* Device B (receiver) processes registration request, making sure that PIN is correct and nonce has not been seen
* Finalise registration. Mutual TLS has been established, proceed to *4.1 Prepare upload*

When receiver cannot scan sender QR codes (broken screen/camera or comms with desktop):

* Device A (sender) sends register request payload outlined in *3.2- Initial Registration*
* Device A (sender) displays the hash of their TLS certificate
* Device B (receiver) processes registration request, making sure that PIN is correct and nonce has not
  been seen (**Note:** this must happen before extracting certificate information in next step)
* Device B (receiver) extracts the sender's certificate information from the connection info of the registration request, hashes it, and displays the Sender Certificate Hash
* Device A (sender) and Device B (receiver) visually compare the hashes as outlined in *2.3- Verification screen*
* If verification succeds, Device B (receiver) pins the Sender Certificate Hash
* Finalise registration. Mutual TLS has been established, proceed to *4.1 Prepare upload*

After Device B (receiver) has pinned the Sender Certificate Hash, Device B
(receiver) will compute each certificate hash on all future requests. For each
request sent after register, Device B (receiver) hashes the certificate information from
the connection and checks the computed hash against the pinned Sender Certificate Hash.

##### Connection scenarios

Connection establishment and exchange of certification information can be done
through a mix of QR code scanning and manual verification of connection
information.  This creates the following four scenarios:

* A. 100% QR flow
* B. Receiver QR code + Sender Hash Verification
* C. Sender QR code + Receiver Hash Verification 
* D. Receiver Hash Verification + Sender Hash Verification (0% QR codes)

**A: Receiver can scan / Sender can scan (100% QR flow)**

1. Receiver scans Sender QR Code (no requests have been sent). Receiver pins Sender Certificate Hash.
2. Sender scans Receiver QR Code (still no requests have been sent). Sender pins Receiver Certificate Hash.
   * mTLS is now established
3. Sender sends `POST /api/v2/register` (this is request 1)

**B: Receiver cannot scan / Sender can scan flow (Sender Hash Verification)**

1. Receiver marks that they can't scan QR codes
2. Sender scans Receiver QR Code (no requests have been sent). Sender pins Receiver Certificate Hash.
3. Sender sends `POST /api/v2/register` (this is request 1)
4. Receiver confirms PIN correct and nonce unseen. Receiver extracts Sender Certificate Hash from register request.
5. Sender Hash Verification commences
6. Both parties confirm Sender Hash and continue. Receiver pins Sender Certificate Hash.
   * mTLS is now established
7. Receiver sends register response back to sender

**C: Receiver can scan / Sender cannot scan flow (Receiver Hash Verification)**

1. Receiver scans Sender QR Code (no requests have been sent). Receiver pins Sender Certificate Hash.
2. Sender does manual connection, types in Receiver IP Address, PIN, Port. Sender sends `POST /api/v2/ping` (this is request 1)
3. Receiver sends back ping response. 
4. Sender extracts Receiver Certificate Hash from ping response.
5. Receiver Hash Verification commences
6. Both parties confirm Receiver Hash and continue. Sender pins Receiver Certificate Hash.
   *  mTLS is now established
7. Sender sends `POST /api/v2/register` (this is request 2)

**D: Receiver cannot scan / Sender cannot scan flow (Receiver Hash Verification and Sender Hash Verification)**:

1. Receiver marks that they can't scan QR codes
2. Sender does manual connection, types in Receiver IP Address, PIN, Port. Sender sends `POST /api/v2/ping` (this is request 1)
3. Receiver sends back ping response. 
4. Sender extracts Receiver Certificate Hash from response.
5. Receiver Hash Verification commences
6. Both parties confirm Receiver Hash and continue. Sender pins Receiver Certificate Hash.
8. Sender sends `POST /api/v2/register` (this is request 2)
9. Receiver confirms PIN correct and nonce unseen. Receiver extracts Sender Certificate Hash from register request.
10. Sender Hash Verification commences
11. Both parties confirm Sender Hash and continue. Receiver pins Sender Certificate Hash.
12. Receiver sends register response back to sender
	* mTLS is now established

### Old Flow (section to be removed after new flow is finalised)

**QR Code Method:**

- Device A (sender) scans the QR code containing:
    - Device B's IP address
    - Port
    - Receiver Certificate Hash
    - PIN
- Device A (sender) pins Receiver Certificate Hash
- Device A (sender) sends a payload to `/api/v2/register` containing:
    - PIN
    - Nonce
- Device B (recipient) receives the payload
- Device B (recipient) checks PIN code is valid 
- Device B (recipient) returns the `sessionId`

After Device A (sender) has pinned Receiver Certificate Hash from the QR code,
Device A (sender) will independently compute each certificate hash on future responses
sent from Device B (recipient). For each response sent by Device B (recipient),
Device A (sender) hashes the certificate from the connection and checks the
computed hash against the Receiver Certificate Hash pinned from the QR code.

**Manual Method:**

Initial Ping:
- Device A (sender) manually types the IP address, PIN, and port
- Device A (sender) sends a ping to `/api/v2/ping`
- Device A (sender) retrieves the Receiver Certificate Hash from recipient 
- Device A (sender) displays the Receiver Certificate Hash to be compared  
- Device B (recipient) displays the Receiver Certificate Hash upon receiving the `/api/v2/ping` request
    
Initial Registration:
- After confirming the Receiver Certificate Hash, Device A (sender) sends the payload to `/api/v2/register`
    - PIN
    - Nonce
- Device B (recipient) receives the payload
- Device B (recipient) checks PIN code is valid
- Device B (recipient) returns the `sessionId`

## 4- File Transfer

### 4.1 Prepare Upload

This request contains only metadata. The receiver decides whether to accept or reject the request.

`POST /api/v2/prepare-upload`

Request Payload

```json5
{
  "title": "Title of the report",
  "sessionId": "uuid-session-identifier",
  "nonce": "random-uuid-number"
  "files": [
    {
      "id": "file-uuid",
      "fileName": "document.pdf",
      "size": 324242,
      "sha256": "57bb905d0f2ccecbb9d81d40daa17e1e05b109c833ddc766edb0b59561088f20",
      "fileType": "application/pdf",
      "thumbnail": "thumbnail-data"
    }
  ]
}
```

Response Payload

```json5
{
  "files": [
    {
      "id": "file-uuid",
      "transmissionId": "uuid-transmission-identifier"
    }
  ]
}
```

**Note:** 

1. `sha256` should be the SHA256 hash of the given file, encoded as a hexadecimal (base 16) string.

Errors:

|HTTP code|Message|
|--|--|
|400|Invalid request format|
|401|Invalid session ID|
|403|Rejected|
|413|Content too large|
|429|Too many requests|
|500|Server error|

### 4.2 File Upload

The file upload requires the sessionId, fileId, and its file-specific transmissionId obtained from /prepare-upload.

`PUT /api/v2/upload?sessionId=sessionId&fileId=fileId&transmissionId=transmissionId&nonce=random-uuid-number`

Request payload

```
raw-binary-data

```

Response payload

```json5
{
  "success": true
}
```

Errors:

|HTTP code|Message|
|--|--|
|400|Missing required parameters|
|401|Invalid session ID|
|403|Invalid transmission ID|
|409|Transfer already completed|
|413|Content too large|
|429|Too many requests|
|500|Server error|
|507|Insufficient storage space|

**Note**: 
1. After a successful upload, the transmissionId should be regarded as used. Any following requests for that transmissionId should return 403 "Invalid tranmission ID".
2. `nonce` should be a unique nonce (UUID V4) for each upload request and tied to each session. See section **5.2 Replay Protection**.

### 4.3 Close Connection

This request is sent by the sender to terminate the session.

The `sessionId` is obtained from /prepare-upload.

`POST /api/v2/close-connection`

Request Payload

```json5
{
  "sessionId": "uuid-session-identifier"
}
```

Response:

```json5
{
  "success": true
}
```

Errors:

|HTTP code|Message|
|--|--|
|400|Invalid request format|
|401|Invalid session ID|
|403|Session already closed|
|429|Too many requests|
|500|Server error|

## 5. Rate-limiting and replay protection

### 5.1 Rate-limiting

All routes are rate-limited per IP address, limiting the amount of requests a
single IP is allowed to make for each route.

If an IP address becomes rate-limited, its request contents should be ignored
and the error 429 "Too many requests" sent as response.

### 5.2 Replay Protection

All routes that submit data during a session are guarded against replay attacks
by the inclusion of a nonce. Nonces are associated with a particular transfer
session. 

A request whose `sessionID` does not match the session ID of an ongoing transfer
session is an invalid request and should be rejected. A request containing a
nonce that has already been handled in an ongoing transfer session is regarded
as invalid and should be rejected.

The above can be modeled as the following pseudocode:

```
// `seen` is a map operated by the receiver with string keys and boolean values
if !sessionValid(request.sessionID) || seen[request.nonce] {
    reject(request)
}
``` 

## 6. Version negotiation

Offline peer-to-peer protocols provide unique opportunities for enabling long-lasting
applications. This presents a difficulty in terms of navigating incompatible versions. 

Protocol version 2 (this document) is wholly incompatible with version 1 due to the
introduction of compulsory mTLS.

### Handling version 1 and version 2 incompatibility

When a receiver or a sender on version 2 detect an incompatibility, they should display the
Incompatible Versions Message describing that the two clients can't proceed because the other
client is running an older, incompatible version.  When to display the Incompatible Versions
Message is described below.

For describing senders and receivers on different version, we use the following notation:

* A *sender on protocol version 2* is titled Sender-v2
* A *receiver on protocol version 1* is titled Receiver-v1

Version 1 and version 2 incompatibility can be detected in the following situations and should be
reacted to as described below. 

1. Sender-v2 scans a version 1 Receiver QR Code. The sender can detect the situation by the
   lack of `protocol_version` in the Receiver QR Code.
	* Sender should not send a register request to the receiver. 
    * Sender should instead display the Incompatible Versions Message on the screen with
      information saying that the receiver is running an older, incompatible version.
2. Receiver-v2 receives any requests with the route prefix `/api/v1/`. It should, if
   possible, respond with a suitable error code that is known to be implemented by a Sender-v1. In particular:
    1. Requests for `/api/v1/register` should return `403 Rejected` to Sender-v1. Receiver-v2
       should display the Incompatible Versions Message on its screen, saying that the sender
       was running the older version. This occurs when a Sender-v1 scans a version 2
       Receiver QR Code.
    2. Requests for `/api/v1/ping` should be ignored (error code 429 is not suitable). When
     receiving version 1 ping requests and a mTLS connection has not yet been established,
     Receiver-v2 should display the Incompatible Versions Message.
3. Sender-v2 sends a request to a Receiver-v1. Requests will be sent to routes with the
   prefix `/api/v2`; requests to these routes may not receive any response. This occurs when a
   Sender-v2 tries to manually connect to a Receiver-v1 i.e. it concerns the *Initial Ping*
   used by the manual connection process.
      * TODO: should the handling behaviour be to have a timeout before displaying "no
        response; maybe incompatible version?" how do android/ios handle this in v1?

### How version 2 and future versions handle incompatibilities

**Scanning Receiver QR code**

When a sender scans a Receiver QR code and the value of field `protocol_version` is an
unsupported version the sender should display the Incompatible Versions Message.

**Receiving responses sent to old routes**

Senders running an older protocol version should handle the following error
code that may be sent back as a response from receivers running a newer incompatible
version:

|HTTP code|Message|
|--|--|
|406|Unsupported version|

As documented above in the protocol, this error code may be sent on the following routes:

* `/api/v2/ping`
* `/api/v2/register`

**Responding to unhandled routes**

When responding to POST requests sent to any unhandled routes, including
unsupported version paths like `/api/v100/ping`, the receiver should send the
following response:

|HTTP code|Message|
|--|--|
|404|Not found|

