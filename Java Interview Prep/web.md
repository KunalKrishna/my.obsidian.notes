
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


#### HTTP Status Code 1xx -5xx

![[Pasted image 20260210191128.png]]
![[HTTP.code.png]]

|Code|Category|**I S R C S**|Description|
|---|---|---|---|
|**1xx**|**I**nformational|**I**|Request received, continuing process.|
|**2xx**|**S**uccess|**S**|The action was successfully received and accepted.|
|**3xx**|**R**edirection|**R**|Further action is needed to complete the request.|
|**4xx**|**C**lient Error|**C**|The request contains bad syntax or cannot be fulfilled.|
|**5xx**|**S**erver Error|**S**|The server failed to fulfill a valid request.|

**I** **S**till **R**ead **C**ode **S**ometimes 
- **I**nformational 1xx
	- **S**uccess 2xx
		- **R**edirection 3xx
			- **C**lient Error 4xx
				- **S**erver Error 5xx 

```
1xx Informational	 
100  Continue
101  Switching protocols
102  Processing
103  Early Hints
 	 
2xx Succesful	 
200	OK
201	Created
202	Accepted
203 Non-Authoritative Information
204	No Content
205	Reset Content
206	Partial Content
207	Multi-Status
208	Already Reported
226	IM Used
 	 
3xx Redirection	 
300	Multiple Choices
301	Moved Permanently
302	Found (Previously "Moved Temporarily")
303	See Other
304	Not Modified
305	Use Proxy
306	Switch Proxy
307	Temporary Redirect
308	Permanent Redirect
 	 
4xx Client Error	 
400	Bad Request
401	Unauthorized
402	Payment Required
403	Forbidden
404	Not Found
405	Method Not Allowed
406	Not Acceptable
407	Proxy Authentication Required
408	Request Timeout
409	Conflict
410	Gone
411	Length Required
412	Precondition Failed
413	Payload Too Large
414	URI Too Long
415	Unsupported Media Type
416	Range Not Satisfiable
417	Expectation Failed
418	I'm a Teapot
421	Misdirected Request
422	Unprocessable Entity
423	Locked
424	Failed Dependency
425	Too Early
426	Upgrade Required
428	Precondition Required
429	Too Many Requests
431	Request Header Fields Too Large
451	Unavailable For Legal Reasons
 	 
5xx Server Error	 
500	Internal Server Error
501	Not Implemented
502	Bad Gateway
503	Service Unavailable
504	Gateway Timeout
505	HTTP Version Not Supported
506	Variant Also Negotiates
507	Insufficient Storage
508	Loop Detected
510	Not Extended
511	Network Authentication Required
```