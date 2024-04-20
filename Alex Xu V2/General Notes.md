# Rest API


Don't use verbs in the URL name, not getAllProducts, just Products with a POST request

I shoudl mention that I've developed my own APIs
* AWS Webhooks
* Returning the proper status codes, organization
* Versioning
	* v1 and v2

# Cache

Kafka can cache on disk

Redis
Elastic Search for inverted Index

SQL has some caching too for frequent things

# OSI

Physical Layer -> Ethernet connections

Data Link Layer -> Headers and Data and TCP ehaders

Network Layer -> IP

Transport Layer -> TCP vs. UDP
* TCP -> Consistent connection, divides data into segments
* UDP -> video streaming, just sends, faster

Session Layer


# API Gateway

Rate limiting
Allow-deny list
Authentication
Parameter validation

Service discovery

# API Architecture Styles

SOAP
* Financial services, where security is important
* Comple
REST
* Easy to use, common
GraphQL
* Only ask for what you need, no over or under
* \Steep learning curve and CPU
Grpc
* Limited browser support
Web Socket
* Real time conneciton
WebHook
* Event driven
* Can notify other machines
* Bad for immediate resposne

# API Testing

Smoke testing
* quick test
*
Functional testing
* detailed testing

Integration testing
* That it works with all the external services and APIs*

Regression testin
* see old broken effecrts

Load testing
	Stress testing
Security testing
Fuzz testing

---
