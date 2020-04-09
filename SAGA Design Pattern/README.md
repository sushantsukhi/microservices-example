# SAGA Design Pattern
SAGA means  
			 _*a long, involved story, account, or series of incidents or story of heroic achievement.*_

**Context**

You have applied the Database per Service pattern. Each service has its own database. Some business transactions, however, span multiple service so you need a mechanism to ensure data consistency across services. 
For example, lets imagine that you are building an e-commerce store where customers have a credit limit. The application must ensure that a new order will not exceed the customer’s credit limit. Since Orders and Customers are in different databases the application cannot simply use a local ACID transaction.

**Problem**

How to maintain data consistency across services?

**Solution**

Implement each business transaction that spans multiple services as a saga. A saga is a sequence of local transactions. Each local transaction updates the database and publishes a message or event to trigger the next local transaction in the saga. If a local transaction fails because it violates a business rule then the saga executes a series of compensating transactions that undo the changes that were made by the preceding local transactions.

Order Service (Loal Transaction) ---> Customer Service (Loal Transaction) ---> Order Service (Loal Transaction)

**There are two ways of coordination sagas:**

### Choreography (Decentralized way)- each local transaction publishes domain events that trigger local transactions in other services

	We want to avoid dependencies in a microservice architecture. So, that each service can work independently. Choreography solves this issue which was the main challenge in orchestration approach.
	In Choreography, every microservice performs their actions independently. It does not require any instructions. It is like the decentralized way of broadcasting data known as events. The services which are interested in those events, will use it and perform actions. This is also known as reactive architecture. The services know what to react to, and how-to, which is more like an asynchronous approach.

	Benefits
		Enables fast processing. As no dependency on the central controller.
		Easy to add and update. The services can be removed or added anytime from the event stream.
		As the control is distributed, there is no single point failure.
		Works well with the agile delivery model, as teams work on certain services rather than on entire application.
		Several patterns can be used. For example, Event sourcing, where events are stored, and it enables event replay. Command-query responsibility segregation (CQRS) can separate read and write activities.
	Limitations
		Complexity is the concerning issue. Each service is independent to identify the logic and react based on the logic provided from the event stream.
		The Business process is spread out, making it difficult to maintain and manage the overall process.
		Mindset change is a must in the asynchronous approach.
	
### Orchestration (Centralized way)- an orchestrator (object) tells the participants what local transactions to execute.
	
	Similarly, in the microservice orchestration, the orchestrator (central controller) handles all the microservice interactions. It transmits the events and responds to it.
	The microservice Orchestration is more like a centralized service. It calls one service and waits for the response before calling the next service. This follows a request/response type paradigm.
	
	Benefits
		Easy to maintain and manage as we unify the business process at the center.
		In synchronous processing, the flow of the application coordinates efficiently.
	Limitations
		Creating dependency due to coupled services. For example, if service A is down, service B will never be called.
		The orchestrator has the sole responsibility. If it goes down, the processing stops and the application fails.
		We lose visibility since we are not tracking business process.

Real Life Example : 

	Choreography - 

		An e-commerce application that uses this approach would create an order using a choreography-based saga that consists of the following steps:

		The Order Service receives the POST /orders request and creates an Order in a PENDING state
		It then emits an Order Created event
		The Customer Service’s event handler attempts to reserve credit
		It then emits an event indicating the outcome
		The OrderService’s event handler either approves or rejects the Order

	Orchestration - 
		
		An e-commerce application that uses this approach would create an order using an orchestration-based saga that consists of the following steps:

		The Order Service receives the POST /orders request and creates the Create Order saga orchestrator
		The saga orchestrator creates an Order in the PENDING state
		It then sends a Reserve Credit command to the Customer Service
		The Customer Service attempts to reserve credit
		It then sends back a reply message indicating the outcome
		The saga orchestrator either approves or rejects the Order


Most of the times, these approaches don’t work well in architecture. So, what is the solution in these use cases?

**Hybrid:**
	Hybrid is the combination of the orchestration approach and choreography. In this approach, we use orchestration within the services whereas we use choreography in between the services.

  Benefits
    The Overall flow is distributed. Each service contains its flow logic.
    Services are decoupled (but to an extent).
  Limitations
    The coordinator is coupled with the services.
    If the coordinator goes down, it impacts the entire system.
    In short, all the approaches have their benefits and trade-offs.

In a Nutshell
	Orchestration uses a centralized approach to execute the decisions and is more crystal clear and has better control. 
	However, Choreography gives more freedom to execute those decisions. 
	On the other hand, we can use both by adopting a hybrid approach to get better results.

So, you can choose from these approaches according to the demands of your architecture.

Reference: 

  This site README was written by making use of below sites -
	1. [microservices.io](https://microservices.io/patterns/data/saga.html)
	2. [softobiz-microservice](https://www.softobiz.com/microservice-orchestration-vs-choreography/)
