## TLDR
* Store status and transaction meta data
* Use Reconciliation to make sure results are same
* Idempotency
* Don't want to store credit card details
# Problem
A system used to settle financial transactions. 

Seem simple but one mistake can cause big revenue loss and reputation damage.

# Step 1 - Understand Problem and Establish Design Scope

##### Need to narrow the requirements. Don't know if interviewer means a digital wallet or something like Stripe / PayPal

#### Candidate Questions

(Can users contribute to multiple charities with a single payment???)

What kind of payment system are we building?
* Payment backend for e-commerece application like Amazon.com
* It handles everything related to money movement

What payment options are supported? Credit cards? PayPal? Bank cards?
* Payment should support all of these in real life. But we can focus on credit cards for now

Do we handle credit card processing ourselves?
* No, we use a third party

Do we store credit card data in our system?
* We do not store card numbers directly in our system

Is the application global? Do we need to support different currencies and international payments?
* Yes

How many transactions per day?
* 1M

Do we need to support the pay-out flow, which an e-commerece site like Amazon uses to pay sellers every month?
* Yes we need to support that

I think I've gathered all requirements, is there anything else I should pay attention to?
* Yes, we need to perform reconciliation and fix any inconsistencies


#### Functional Requirements

Pay-in flow -> Payment system receives money form customers on behalf of sellers

Pay-out flow -> Payment System sends money to sellers around the world.

### Non-Functional Requirements

Reliability and Fault Tolerance
* No duplicates or missed charges

Reconciliation process between internal services

### Back of Envelope Estimations

1M transactions a day -> 1 10 ^ 6 / 10^5 -> 10 QPS
* Really small QPS, could handle this with one database

Focus of this system design isn't high throughput, it's handling the payment transactions

# Step 2 - Propose High-Level Design and Get Buy in

![[Pasted image 20240218092608.png]]

In-Flow
![[Pasted image 20240218092643.png]]

Payment Service
* Accepts payment events from users and coordinates the payment process
* Does a risk check using a third-party provider
*  We accept multiple payment orders for different sellers

Payment Executer
* Executes a single payment order via Payment Service Provider (Stripe, Paypal)
*
PSP
* Moves money from account A to Account B

Card Schemes
* Card Schemes are the organizations that process credit card operations
* Visa, Mastercard, Discovery, etc.

Ledger
* Keeps financial records for post-payment analysis
* Say X paid Y with Z amount
* Important for calculating reveneue or forecasting
* Maybe even reconcilliation?

Wallet
* Account balance of the merchant
* Can show how much a user has paid in total

#### APIs for payment service

POST /v1/payments
* buyer_info
* checkout_id
* credit_card_info
	* Encrypted
* payment_orders
	* Seller account
	* amount
		* Use string for amount, not double
		* Double can be broken depending on number serialization
			* Also has a max limit of a number around 10^9?
	* currency
	* payment order_id

GET /v1/payments/{:id}
* Check status of a payment

##### Data model for payment service

Most likely going to do SQL at 10 QPS
* We gain ACID properties too

We want
* Proven stability
* Richness of supporting tools like monitoring and investigation
* A database with lots of talent / job market to manage

Xu recommends SQL

###### Payment Event table

![[Pasted image 20240218093837.png]]

##### Payment Order Table

![[Pasted image 20240218093903.png]]

Checkout creates a payment event that can contains several payment orders to process

After sending to PSP, we dont' directly transfer money to seller. We hold it in the e-commerce website's bank account for "pay-in" and distribute it when pay-out condition is satisfied

Use the `payment_order_status` to track where we're at
* Start at NOT_STARTED
* Once it's sent to PSE, we do EXECUTING
* Then do, SUCCESS or DENIED depending on PSE's response

SUCCESS means we start the wallet process
And then the ledger process and update it's field (if database gives an ACK, I guess)

#### Double-Entry Ledger System

![[Pasted image 20240218094539.png]]

Sum of transactions must be $0

Proves end-to-end traceability

### Hosted Payment Page
* Most companies do not store credit card info because they need to deal with regulations like Payment Card Industry Data Security Standard (PCI DSS) in USA
* Websites, Iframes, or SDKs are available that will store the card info so you don't have ot
#### Pay out flow
* Similar to pay-in-flow
* Just take the money from the ecommerce website bank account / wallet

# Step 3 - Design Deep Dive

Focus on making the system more robust, faster, and secure
### PSP Integration

Visa and Mastercard only allow integration directly for massive companies that can afford the investment on their part (It's a lot of specific work)

Other companies have to do
* Safely storing credit card info and then using an API
	* PSP will call their API to retrieve the customer info to charge
* If company doesn't store it, PSP offer a hosted payment page it collect the card info themselves

![[Pasted image 20240218095442.png]]

#### Reconciliation

Periodically compares states among related services to verify they are in agreement.

The last line of defense of payment system.

Every night the PSP or banks send settlement file to their clients. 

Finance team usually performes manual adjustments if discrepancy occurs

Three types of mismatches
* Mismatch is classifiable and adjustment is automated
* Mismatch is classifiable and can't automate the adjustment
* Mismatch is unclassifiable. Placed in a special job queue for finance team to investigate

### Handling Payment Process Delays

PSP can take forever to get back or wants a high investigation
Credit Card requires extra protection and details from the card holder

PSP would return pending status to the client and our client would display that to the user
PSP tracks pending payment on our behalf and notifies payment service via webhook that it's pending


#### Communication among internal services

Synchronous HTTP doesn't work well as scale increases. Long request and repsonse cycle increaes latency

Asynchronous 
* Message queues using producers and consumers

![[Pasted image 20240218100352.png]]

#### Handling Failed Payments

Track the payment state
* can retry pased on the last state of the payment
Retry queue and dead letter queue
* repeated failures land in the dead letter queue

![[Pasted image 20240218100509.png]]


#### Exactly-Once Delivery

need to make sure a retry doesn't end up succeeding
* we can wait a fixed amount of time
* Or incremental intervals
* Or cancel the request

#### Dont' allow user to click pay button quikcly twice

Use a cache to make sure the request is fresh / new

# Skipping the last 2 pages of this Chapter to come back later


 ![[Pasted image 20240218100851.png]]

