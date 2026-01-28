
```xml
<dependency> 
	<groupId>org.springframework.boot</groupId
	<artifactId>spring-boot-starter-security</artifactId>
<dependency>
```

**Authentication** - identifies **who** the user is. 
* = **prove** or show (something) to be true, genuine, or valid.
* Microsoft Authenticator
**Authorization** - defines **what** the user is allowed to do.
* = give official **permission** for or approval to (an undertaking  or agent).



# JWT (pronounced “jot”) 
JSON Web Token

Goal : 
- to build a **stateless** authentication / authorization flow 
- that is easily scalable and **eliminates the need for ~~servers~~** to maintain session information.
##  How Does JWT Work?

Let's look at the flow:
1. **User logs in.**
2. **Server checks credentials.** If ok, the server creates a JWT containing the user info (e.g., username, roles).
3. **Server signs the JWT** with a secret (HMAC) or private key (RSA).
4. **Server sends JWT back** to client (browser, mobile app).
5. **Client stores JWT**, usually in memory or local storage.
6. **For each request**, the client sends JWT (typically in the HTTP Authorization header).
7. **Server checks JWT**:
    - Is the signature valid and not expired?
    - If yes, server trusts the user info inside the token, and doesn't need to check the database each time

## JWT Structure
A JWT is three Base64-**encoded (~~not encrypted~~ ) strings** joined by dots:
```JSON
<Header>.<Payload>.<Signature>
<JSON object>.<JSON object>.<Signature>
```
- **Header:** Info about the token (type and signing algorithm).
- **Payload:** Data to transmit (like user id, roles). This is called "claims".
- **Signature:** Used to verify the token was not changed.

Base64Url encoding : a way to convert binary/text data into a format that is safe to transmit in URLs and web requests.
- **Base64** uses: `A–Z`, `a–z`, `0–9`, `+`, `/` 
- **Base64Url** uses: `A–Z`, `a–z`, `0–9`, `-`, `_` 

### **Header** : 
JSON object
1. encryption algo
2. token type 
```JSON
{ 
	"alg": "HS256", 
	"typ": "JWT" 
}
```

###  **Payload** : 
JSON object where all of the transmitted data lives. Also called a _claim_, this data typically contains 
* **user information** (username, email address), 
* **session data** (IP address, time or last login), or 
* **authorization permissions** (roles or groups the user belongs to).
Payload includes claims like subject, expiration, and issue time. : 
- expiration time (exp), 
- issued at (iat), 
- not before (nbf)
```JSON
{ 
	"drn": "DS", 
	"exp": 1680902696, 
	"rexp": "2023-05-05T21:14:56Z" 
}
```

###  **Signature** : 
created by signing the Base64Url encoded **header** and **payload** with a **secret key** and an **algorithm** specified by the developers. It is used to verify that the sender of the JWT is who they claim to be and ensure the token's integrity.

To create the signature part you have to take the **encoded header**, the **encoded payload**, **a secret**, the **algorithm** specified in the header, and sign that.

```css
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)

```

 use [jwt.io Debugger](https://www.jwt.io/) to decode, verify, and generate JWTs.

**What is a Digital Signature?**

A **digital signature** is a cryptographic technique that allows someone to prove a piece of data (like a message or document) was 
- created by them and 
- hasn’t been tampered with.

**Key points:**
- Uses mathematical algorithms (like RSA, ECDSA, etc.).
- Involves a **private key** (kept secret by signer) and a **public key** (shared with anyone).
- Anyone with the public key can check if a signature is genuine and that the data wasn't altered.

**How it works:**
1. The sender creates a _hash_ of the data (a unique fingerprint).
2. The sender **encrypts** this hash with their _private key_ (this produces the digital signature).
3. The receiver gets the data and the signature.
4. The receiver:
    - Uses the sender’s _public key_ to decrypt the signature (recovering the hash sent).
    - Hashes the original data themselves.
    - Compares the two hashes. If they match, the signature is valid and the data is unchanged.

**Digital Signatures in JWTs**

A JWT (JSON Web Token) consists of three parts: header, payload, and signature.
- The **signature** is a digital signature.
- It's created by the issuer (the server or auth provider) when generating the token.

**Role and process:**
- The issuer:
    1. Base64Url-encodes the JWT header and payload.
    2. Signs this data using a secret key (HMAC) or a private key (RSA/ECDSA), creating the signature.
- The consumer (like another server or an API):
    1. Separates the JWT into header, payload, and signature.
    2. Re-creates the signature locally (using the secret or public key).
    3. Checks if the locally created signature matches the JWT's signature.

**Purpose in JWTs:**
- **Integrity:** Proves the token hasn’t been changed since it was issued.
- **Authenticity:** Shows the token really came from the trusted issuer.



#### Bibliography : 
[JSON Web Token Introduction - jwt.io](https://www.jwt.io/introduction#how-json-web-tokens-work)
