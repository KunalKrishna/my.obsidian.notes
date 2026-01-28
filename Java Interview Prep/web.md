
## Anatomy of an HTTP Message  : 
[HTTP messages - HTTP \| MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Messages)

![[http basics.png]]
The start-line and headers of the HTTP message are collectively known as the _head_ of the requests, and the part afterwards that contains its content is known as the _body_.

| Head         | Body    |
| ------------ | ------- |
| Request Line | content |
| Header       |         |

### Components of HTTP Request
![Example of headers in an HTTP request](https://mdn.github.io/shared-assets/images/diagrams/http/messages/request-headers.svg)
1. **Request Line** : `<method> <request-target> <protocol>` eg `POST /users HTTP/1.1`
	1. [HTTP method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods) (also known as an _HTTP verb_)
	2. absolute or relative [URL](https://developer.mozilla.org/en-US/docs/Glossary/URL),
	3. _HTTP version_ e.g. `HTTP/1.1`
2. **HTTP Headers**(optional) : *metadata* sent with a request after the start line and before the body
	1. Request Headers
	2. Representation Headers
	3. ```HTML
	   Host: example.com
	   User-Agent: curl/8.6.0
	   Accept: */*
	   Content-Type: application/x-www-form-urlencoded
	   Content-Length: 49
	   ```
3. **Empty line** : ------------indicating the metadata of the message is complete------------
4. **Body(optional)** : the part of a request that carries information to the server
	1. Only `PATCH`, `POST`, and `PUT` requests have a body


### Components of HTTP Response
Responses are the HTTP messages a server sends back in reply to a request. The response lets the client know what the outcome of the request was.

![Example of headers in an HTTP response](https://mdn.github.io/shared-assets/images/diagrams/http/messages/response-headers.svg)

1. **Status Line** : `<protocol> <status-code> <reason-phrase>` eg `HTTP/1.1 201 Created`
2. **HTTP Headers**(optional) : *metadata* sent with a response
	1. Response Headers 
	2. Representation Headers 
```HTML
Content-Type: application/json 
Location: http://example.com/users/123
```
3. **Empty line** : ------------indicating the metadata of the message is complete------------
4. **Body(optional)** : the part of a request that carries information to the server
```JSON
	{ "message": "New user created", 
	   "user": { 
		   "id": 123, 
		   "firstName": "Example", 
		   "lastName": "Person", 
		   "email": "bsmth@example.com" 
		   }
	   }
```