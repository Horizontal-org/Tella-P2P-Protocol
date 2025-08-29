# Tella P2P Protocol

A secure, offline peer-to-peer file sharing protocol designed for Tella applications. This protocol enables encrypted file transfers between devices without relying on external servers, prioritizing privacy and security for sensitive data exchange.


## 1- Security Features

* All connections are secured with HTTPS using self-signed certificates generated per device.
* Authentication is mandatory via PIN and IP address, provided through QR code scanning or manual entry.
* Certificates are verified to prevent man-in-the-middle (MITM) attacks
* All connections use a specific port : 53317


## 2- Connection Authentication

All connections require authentication, either via QR code or manually

### 2.1- QR authentication (primary method)

The host device displays a QR code containing:

* Host's IP addresses
* Connection PIN
* Port
* Hash of the tls certificate

QR payload:

```json5
{
  ip_address: String,
  port: Number,
  certificate_hash: String,
  pin: String
}
```

### 2.2- Manual authentication (Fallback Method)

When QR code scanning is not available, the host device will display:

* IP address
* 6 digit PIN
* Port number 


After entering the connection information, both the sender and the receiver will display a verification screen containing an alphanumeric sequence that encodes the hash of the TLS certificate.

Both parties will verify that the same sequence is shown on each device before proceeding.

The verification screen will provide two options:
* Confirm and Connect — Proceed with registration if the hashes match.
* Discard and Start Over — Terminate the connection and return to the initial state.


Example of alphanumeric sequence:

```markdown
87fd 5869 a6b3 e414 112c 1934 ca00 be77 b8e4 584c 829a 4536 490b da9a 3928 be4a
```

**Security Note:** A hash mismatch indicates a potential man-in-the-middle (MITM) attack.
Users should verify that they are connecting to the intended device and ensure the network environment is secure before retrying.

## 3- Connection Establishment

### 3.1 Initial Ping

`POST /api/v1/ping`

This endpoint initiates a secure handshake between two devices during the manual connection process. It must be called before the register endpoint. Once called, both the sender and receiver display the verification screen.

### 3.2- Initial Registration

For QR code authentication, it is performed immediately after the QR code has been scanned.

For manual authentication, it is performed after the ping request and once the sender has verified the certificate hash.

`POST /api/v1/register`

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

**Note:** A maximum of 3 invalid requests are allowed

Errors:

|HTTP code|Message|
|--|--|
|400|Invalid request format|
|401|Invalid PIN|
|403|Rejected| 
|409|Active session already exists|
|429|Too many requests|
|500|Server error|

### Flow

Let’s consider Device A as the sender and Device B as the recipient

**QR Code Method:**

- Device A (sender) scans the QR code containing:
    - Device B's IP address
    - PIN
    - Port
    - Certificate Hash
- Device A (sender) sends the payload to `/api/v1/register`
- Device B (recipient) receives the payload
- Device B (recipient) returns the `sessionId`

The Certificate Hash from the QR code is automatically compared with the hash received from the server.

**Manual Method:**

Initial Ping:
- Device A (sender) manually types the IP address, PIN, and PORT
- Device A (sender) sends a ping to `/api/v1/ping`
- Device A (sender) retrieves the Certificate Hash from server 
- Device A (sender) displays the Certificate Hash to be compared  
- Device B (recipient) displays the Certificate Hash upon receiving the `/api/v1/ping` request     
    
Initial Registration:
- After confirming the Certificate Hash, Device A (sender) sends the payload to `/api/v1/register`
- Device B (recipient) receives the payload
- Device B (recipient) confirms the registration request     
- Device B (recipient) returns the `sessionId`


## 4- File Transfer

### 4.1 Prepare Upload

This request contains only metadata. The receiver decides whether to accept or reject the request

`POST /api/v1/prepare-upload`

Request Payload

```json5
{
  "title": "Title of the report",
  "sessionId": "uuid-session-identifier",
  "files": [
    {
      "id": "file-uuid",
      "fileName": "document.pdf",
      "size": 324242,
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

Errors:

|HTTP code|Message|
|--|--|
|400|Invalid request format|
|401|Invalid session ID|
|403|Rejected|
|500|Server error|

### 4.2 File Upload

The file upload requires the sessionId, fileId, and its file-specific transmissionId obtained from /prepare-upload.

`PUT /api/v1/upload?sessionId=sessionId&fileId=fileId&transmissionId=transmissionId`


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
|500|Server error|

### 4.3 Close Connection

This request is sent by the sender to terminate the session.

The `sessionId` is obtained from /prepare-upload.

`POST /api/v1/close-connection`

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
|400|Missing session ID|
|401|Invalid session ID|
|403|Session already closed|
|500|Server error|

