# Tella P2P Protocol

A secure, offline peer-to-peer file sharing protocol designed for Tella applications. This protocol enables encrypted file transfers between devices without relying on external servers, prioritizing privacy and security for sensitive data exchange.


## 1- Security Features

* All connections use HTTPS with per-device generated self-signed certificates
* Mandatory authentication via PIN and connect code or QR code scanning
* verification of certificate to prevent MITM attacks

## 2- Defaults

**HTTP/HTTPS (TCP)**

* Default Port: 53317
* Alternative ports: User configurable if default is unavailable

## 3- Connection Authentication

All connections require authentication via one of these methods:

### 3.1- QR authentication (primary method)

The host displays a QR code containing:

* Host's full list of local IP addresses
* Connection PIN
* port
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

### 3.2- Manual Authentication (Fallback Method)

For scenarios where QR scanning isn't possible:

**Host displays:**

* IP address
* 6 digit PIN
* Port number (format XXX)

After entering the information, both the sender and receiver will show a verification screen including an alphanumeric sequence (encoding the hash of the tls certificate) and will be prompted to verify if the same sequence is shown in both sides. There will be 2 buttons

* confirm and connect - Proceeds with registration if the hashes match
* discard and start over - terminates connection and returns to initial state

Example of alphanumeric sequence:

```markdown
87fd 5869 a6b3 e414 112c 1934 ca00 be77 b8e4 584c 829a 4536 490b da9a 3928 be4a
```

Security Note: Hash missmatch indicate a potential man-in-the-middle attack. Users should verify they are connecting to the correct device and check network security before retrying

## 4- Connection Establishment

### 4.1 Initial Ping

`POST /api/v1/ping`

used before the register to perform a secure handshake between two devices.

### 4.2- Initial Registration

should perform instantly after the ping request

`POST /api/v1/register`

Request payload

```markdown
{
  pin: "123456",
  nonce: "random-uuid-number",
}
```


response payload

```markdown
{
  "sessionId": "uuid-session-identifier"
}
```

**note: **We should implement a rate limit for the invalid, probably 3 request

**errors**:

|HTTP code|Message|
|--|--|
|400|Invalid request format|
|401|Invalid PIN|
|403|Invalid encryption/decryption|
|409|Active session already exists|
|429|Too many requests|
|500|Server error|

#### flow

**QR code:**

- Device B scans QR code with:
    - Device's A IP address
    - PIN
- Device B (sender) sends a ping to `/api/v1/ping`
- handshake is executed between device B and device A
- Device B (sender)  sends the payload to `/api/v1/register`
- Device A (recipient)  receives payload
- Device A returns the `uuid-session-identifier`

During this handshake we compare automatically the hashes


**Manual Method:**

- Device B Type manually the ip address, PIN and PORT
- proceeds with normal registration
- we will prompt on both devices the hashes and let the users compare it manually

## 5- File Transfer

### Prepare Upload

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
      "sha256": "file-hash"
    }
  ]
}
```

Response Payload

```json5
{
  "transmissionId": "uuid-transmission-identifier"
}
```

**errors:**

|HTTP code|Message|
|--|--|
|400|Invalid request format|
|401|Invalid session ID|
|403|Rejected|
|500|Server error|

### File Upload

`POST /api/v1/upload`

Request payload

```json5
{
  "sessionId": "uuid-session-identifier",
  "transmissionId": "uuid-transmission-identifier",
  "fileId": "file-uuid",
  "data": "raw-binary-data"
}
```

Response payload

```json5
{
  "success": true
}
```

errors:

|HTTP code|Message|
|--|--|
|400|Missing required parameters|
|401|Invalid session ID|
|403|Invalid transmission ID|
|409|Transfer already completed|
|413|File too large|
|415|Unsupported file type|
|507|Insufficient storage space|
|500|Server error|

### Close Connection

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

errors:

|HTTP code|Message|
|--|--|
|400|Missing session ID|
|401|Invalid session ID|
|403|Session already closed|
|500|Server error|
````
