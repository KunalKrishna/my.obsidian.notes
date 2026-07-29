
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


# Why JWT ?

**JWT's primary purpose is Authentication persistence** — proving "I already proved who I am, here's my proof" on every request, without repeating the login process.

| Your Point                                                 | Correct?        |
| ---------------------------------------------------------- | --------------- |
| Reduces server from verifying username/password every time | ✅ Exactly right |
| Saves user from sending username/password every request    | ✅ Exactly right |
The deeper benefit — **Statelessness**
Traditional sessions still verify you on every request too — but they do it by **looking you up in server memory**:

```
Session approach:
Request → "session_id=abc123" → Server looks up abc123 in memory → finds your user → OK

JWT approach:
Request → "token=eyJhbG..." → Server just does MATH to verify signature → no lookup needed
```

So JWT doesn't just save the **user** from resending credentials — it saves the **server from maintaining any memory at all**. The token is entirely self-contained.

> [!JWT-One Sentence Summary] 
> "Log in once, get a signed pass, show that pass on every visit — the server trusts the pass without remembering you."

## JWT (pronounced “jot”) 
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


---
# OAuth

> **OAuth and JWT are completely different things — but they are often used together.** They solve _different problems_ and operate at _different levels._

The best analogy:

> **OAuth is a PROCESS (a set of rules)** **JWT is a FORMAT (a data structure)**
> 
> One is _how you get access_. The other is _what the access token looks like_.

---

## What Problem Does Each Solve?

| OAuth              | JWT                                                                             |                                                   |
| ------------------ | ------------------------------------------------------------------------------- | ------------------------------------------------- |
| **Solves**         | "How do I let App B access my data on App A, without giving App B my password?" | "How do I carry user info in a request securely?" |
| **Is a**           | Protocol / Framework (a process)                                                | Token format (a data structure)                   |
| **Think of it as** | A valet parking system                                                          | The valet ticket itself                           |



---

## OAuth Explained Simply

Forget code for a moment. Real world scenario:

> You want to use **Spotify** and it asks — _"Login with Google?"_ You click yes, Google asks _"Allow Spotify to see your email and profile?"_ You say yes. Spotify gets access. **You never gave Spotify your Google password.**

**That is OAuth.** It's a protocol that lets one app access your data on another app — **with your permission** — without sharing your password.

### The 4 Players in OAuth

```
YOU  ──────────────────────────────────────────────────────────
(Resource Owner — you own the Google account)

SPOTIFY  ──────────────────────────────────────────────────────
(Client — the app that wants access)

GOOGLE  ───────────────────────────────────────────────────────
(Resource Server — has your data)

GOOGLE AUTH SERVER  ────────────────────────────────────────────
(Authorization Server — issues the access token)
```

### The OAuth Flow Step by Step

```
1. You click "Login with Google" on Spotify

2. Spotify redirects you to Google's login page
   (notice the URL changes to accounts.google.com)

3. Google asks: "Allow Spotify to see your profile?"

4. You click Allow

5. Google gives Spotify an ACCESS TOKEN
   (not your password — just a limited-access token)

6. Spotify uses that token to call Google's API
   and fetch your name/email

7. Done — Spotify never knew your Google password!
```

---

## How JWT Fits Into OAuth

Here's where they connect:

> When Google (Step 5 above) gives Spotify that **access token** — that token is often a **JWT**!

So OAuth says _"here's an access token"_ — and JWT is simply the **format** that token is commonly written in.

```
OAuth says:        "I will give you an access token"
JWT says:          "Here's what that token looks like:
                    eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ..."
```

But — and this is important — **OAuth doesn't require JWT**. OAuth can use other token formats too. And JWT can be used without OAuth (like we did in the Spring Security example earlier — no OAuth involved at all).

---

## Side by Side Comparison

```
WITHOUT OAuth (what we built earlier):
────────────────────────────────────────
User ──► Your App: "here's my username/password"
Your App creates a JWT and sends it back
User sends that JWT on every request
Your App verifies it

= One app, one server, JWT used for session management


WITH OAuth (e.g. Login with Google):
────────────────────────────────────────
User ──► Your App: "login with Google"
Your App ──► Google: "this user wants access"
Google ──► User: "do you allow this?"
User ──► Google: "yes"
Google ──► Your App: here's an access token (often a JWT)
Your App uses that token to talk to Google's API

= Multiple apps involved, delegated authorization
```

---

## The Simplest Way To Remember It

> 🎫 **OAuth** is the _system_ that decides IF you get a ticket and what it grants access to.
> 
> 📄 **JWT** is the _paper_ the ticket is printed on.

You can print the ticket on different paper (other token formats). The paper format (JWT) can also be used outside the ticketing system (OAuth).

---

## Are They Related?

|Statement|True?|
|---|---|
|OAuth and JWT solve the same problem|❌ No — different problems|
|JWT requires OAuth|❌ No — JWT works without OAuth|
|OAuth requires JWT|❌ No — OAuth can use other token formats|
|They are often used together|✅ Yes — OAuth commonly uses JWT as its token format|
|They are completely independent|✅ Yes — independent but complementary|

---

> 💡 **Bottom line:** You already understand JWT from before. OAuth is a completely separate concept about _delegated access between apps_. They happen to work well together, which is why people often mention them in the same breath — but they are not the same thing!







---
#### Bibliography : 
[JSON Web Token Introduction - jwt.io](https://www.jwt.io/introduction#how-json-web-tokens-work)
