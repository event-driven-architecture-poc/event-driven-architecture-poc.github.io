# Project
This project demonstrates the use of an event-driven microservice architecture using event sourcing. The Command Query Responsibility Segregation pattern (CQRS) is not used here. The architecture will be applied to a simplified e-commerce website with functionality outlined below. 
> Note: This is a passion project and implemented for learning purposes. 

## Table of Contents
- [System Overview](#system-overview)
- [Architecture](#architecture)
    + [Components](#components)
    + [Events](#events)
    + [System Flow](#system-flow)
      - [End-to-end Flow](#end-to-end-flow)
      - [Payment Update](#payment-update)
      - [Shipping Address Update](#shipping-address-update)
      - [Order Cancellation](#order-cancellation)
- [FAQ](#faq)


## System Overview
The e-commerce website will allow customers to place orders, view the status of their orders, update payment and shipping details for their order, and cancel their order.

### Order Lifecycle
The lifecycle for a customer's order is as follows:
1. A customer places an order through the website. Once the order has been created, the customer receives an order confirmation email notification.
2. Payment authorization for the full dollar amount of the order total gets made on the customer's credit card. If the authorization fails, the customer receives an email notification indicating that the payment authorization failed.
3. Once the payment gets authorized successfully, the shipping preparation begins.
4. The customer will receive an email notification once their order has shipped out for delivery. Payment also gets captured at this point.

#### Customer Actions
Customers will also be allowed to do the following:
- View the current status of their order
- Update their payment method for the order, given that the existing payment authorization and capture is not in progress or complete, or the order is not in a cancelled state.
- Update their shipping address for the order, given that shipping preparation is not in progress or complete, or the order is not in a cancelled state.
- Cancel their order, given that shipping preparation is not in progress or complete, or the order is not already in a cancelled state. The customer will receive an order cancelled email notification if their order gets cancelled. Additionally, their payment authorization will be voided if it had been authorized previously.

## Architecture

### Components
![Component Diagram](/images/component-diagram.png)

#### API Gateway
The API Gateway acts as the public interface for any front-end requests. It exposes an HTTP REST API with endpoints for submitting orders, getting order details, updating payment, updating shipping details, and cancelling an order.

#### Order Event Service
The Order Event Service is a web service which exposes an HTTP REST API with endpoints for creating events in the event store (and optionally publishing them to an event broker) and deriving the current order state from the event store.

#### Payment Service
The Payment Service is a web service which is responsible for payment processing. It subscribes to various events and publishes various events (through the Order Event Service) in response to various events.

#### Shipping Service
The Shipping Service is a web service which is responsible for any operations related to shipping for an order. It subscribes to various events and publishes various events (through the Order Event Service) in response to various events/requests.

#### Notification Service
The Notification Service is a web service which is responsible for sending out notifications to customers in response to various order events.

#### Event Broker
The event broker (denoted by the blue hexagon in the component diagram above) acts as middleware that provides the infrastructure for routing events. In this case, [RabbitMQ](https://www.rabbitmq.com/) will be used as the event broker.

##### RabbitMQ Components
![Component Diagram](/images/rabbitmq-components.png)

The RabbitMQ topology will be broken down as follows:

###### Exchanges
All exchanges will be declared as topic exchanges. There will be three exchanges:
- `orders` exchange
- `payment` exchange
- `shipping` exchange

###### Routing Keys
Messages published to the exchanges will have routing keys defined with a naming convention of `<category>.<event>`, e.g. `orders.order-created`.

###### Queues
Each consuming application will have its own dedicated queue. In this instance, we will have three queues:
- **payment** queue:
    - The Payment Service will consume events off this queue.
    - Bindings:
        - To the Orders exchange with routing patterns:
            - `orders.order-created`
            - `orders.order-cancelled`
        - To the Payment exchange with routing patterns:
            - `payment.payment-updated`
        - To the Shipping exchange with routing patterns:
            - `shipping.shipped` 
- **shipping** queue:
    - The Shipping Service will consume events off this queue.
    - Bindings:
        - To the Payment exchange with routing patterns:
            - `payment.payment-authorized`
- **notification** queue:
    - The Notification Service will consume events off this queue.
    - Bindings:
        - To the Orders exchange with routing patterns:
            - `orders.order-created`
            - `orders.order-cancelled`
        - To the Payment exchange with routing patterns:
            - `payment.payment-auth-failed`
        - To the Shipping exchange with routing patterns:
            - `shipping.shipped`

###### Message Properties
The following message properties will be used:
- `delivery-mode` - A property that is required by RabbitMQ that indicates whether messages should be persisted to disk before being delivered. `1` is used to indicate a non-persistent message and `2` is used to indicate a persistent message.     
- `content-type` - This will be used by consuming applications as an indicator of the type of payload. In this case it will always be set to `application/json`.
- `message-id` - A unique identifier for the event.
- `correlation-id` - Identifies a related message which contributed to the publishing of this message.
- `app-id` - Identifies the publisher of the message.
- `type` - Identifies the event type. Will be in the form of `<event>/<version>`, e.g. `order-created/v1`
- `timestamp` - The published timestamp of the message.

### Events
- `order-cancelled`
- `order-created`
- `payment-auth-failed`
- `payment-auth-in-progress`
- `payment-auth-voided`
- `payment-authorized`
- `payment-captured`
- `payment-updated`
- `shipped`
- `shipping-address-updated`
- `shipping-preparation-in-progress`
- `shipping-prepared`

### System Flow

#### End-to-end Flow
1. The API Gateway receives a request to create an order.
2. The API Gateway makes an API request to the Order Event Service to create a new order. This request will contain all the order information including payment and the shipping address.
3. The Order Event Service writes an `order-created` event to the event store and publishes the event.
4. The following applications will subscribe to and process the `order-created` event:
    - **Notification Service**: Will send out an order received email to the customer.
    - **Payment Service**: The Payment Service will make an API call to the Order Event Service to write a `payment-auth-in-progress` event to the event store on the condition that the order is not in a cancelled state. Once complete, it will derive the current order state from the Order Event Service and attempt the payment authorization. If the payment authorization is successful, it will make another API call to the Order Event Service to write a `payment-authorized` event to the event store and publish it. However, if the payment authorization fails, it will make an API call to the Order Event Service to write a `payment-auth-failed` event to the event store and publish it. The Order Notification Service would then consume the `payment-auth-failed` event and send out a payment authorization failed email to the customer.
5. The Shipping Service will subscribe to and process the `payment-authorized` event. It will make an API call to the Order Event Service to write a `shipping-preparation-in-progress` event to the event store on the condition that the order is not in a cancelled state. Once complete, it will derive the current order state from the Order Event Service and begin the shipping preparation process. If the shipping preparation is successful, it will make another API call to the Order Event Service to write a `shipping-prepared` event to the event store and publish it.
6. The Shipping Service receives a request indicating the order has been shipped out. It will make an API call to the Order Event Service to write a `shipped` event to the event store and publish it.
7. The following applications will subscribe to and process the `shipped` event:
       - **Notification Service**: Will send out an order shipped email to the customer
       - **Payment Service**: Will attempt a payment capture and then make an API call to the Order Event Service to write a `payment-captured` event to the event store.

#### Payment Update
Payment updates will work as follows: 
1. The API Gateway receives a payment update request.
2. The API Gateway makes an API call to the Order Event Service to update payment. This request will contain the updated payment information.
3. The Order Event Service determines if the payment can be updated according to some business rules. Payment cannot be updated under the following conditions:
    - there is a `payment-auth-in-progress` event in the event store but no succeeding `payment-authorized` or `payment-auth-failed` event in the event store
    - there is a `payment-authorized` event in the event store
    - there is an `order-cancelled` event in the event store
4. If the payment can be updated, the Order Event Service will write a `payment-updated` event to the event store and publish it. If not, an error response will be returned.
5. The Payment Service will subscribe to and process the `payment-updated` event. 
    - If the Payment Service receives an `order-created` event before it receives the `payment-updated` event, then it will discard the `order-created` event. When it receives the `payment-updated` event, it will attempt authorization.
    - If the Payment Service receives a `payment-updated` event before it has received an `order-created` event, then it will simply discard the `payment-updated` event. When it receives the `order-created` event, it will attempt authorization.
    - If the Payment Service had already previously processed an `order-created` event (in which case we know that it would have resulted in a payment authorization failure), then it will attempt authorization once it receives the `payment-updated` event. 
    - If there were multiple payment updates done, only the latest `payment-updated` event will be processed.
    
#### Shipping Address Update
Shipping address updates will work as follows: 
1. The API Gateway receives a shipping address update request.
2. The API Gateway makes an API call to the Order Event Service to update shipping address. This request will contain the updated shipping address information.
3. The Order Event Service determines if the shipping address can be updated according to some business rules. In this instance, shipping address can only be updated if there are no `shipping-preparation-in-progress` or `order-cancelled` events in the event store.
4. If the shipping address can be updated, the Order Event Service will write a `shipping-address-updated` event to the event store. If not, an error response will be returned.
5. When the Shipping Service processes a `payment-authorized` event, it will use the updated shipping address.

#### Order Cancellation
Order cancellations will work as follows: 
1. The API Gateway receives an order cancellation request.
2. The API Gateway makes an API call to the Order Event Service to cancel the order.
3. The Order Event Service determines if the order can be cancelled according to some business rules. In this instance, an order can only be cancelled if there is no `shipping-preparation-in-progress` or `order-cancelled` events in the event store.
4. If the order can be cancelled, the Order Event Service will write an `order-cancelled` event to the event store and publish it. If not, an error response will be returned.
5. The following applications will subscribe to and process the `order-cancelled` event:
    - **Notification Service**: Will send out an order cancelled email to the customer.
    - **Payment Service**: Will void the last authorization if authorization was complete previously. It will then make an API call to the Order Event Service to write a `payment-auth-voided` event to the event store. If the payment was never authorized, then the event will be discarded.

## FAQ

### Q. Why use event sourcing?
Event sourcing provides certain benefits such as providing a log of all the notable changes that are occurring in a system via events with snapshots of the changes. By doing so, we are able to derive state at any point in time and track all updates in isolation. 

### Q. Why is CQRS not used for the event store?
This project doesn't use CQRS for the event store as CQRS would normally add a layer of complexity and introduces eventual consistency into the system which would make dealing with synchronous update operations (payment updates, shipping address updates, order cancellations, etc.) that rely on consistent state much more complex.  

### Q. When are messages removed from a queue in RabbitMQ?
A message gets removed from a queue once the RabbitMQ broker receives an acknowledgement from the consuming application. A consuming application returns an acknowledgement only after it has captured the message and/or processed the message.

### Q. What happens if a consuming application is unable to process an event?
There are essentially three cases to consider as to why a consuming application is unable to process an event:
1. If the event is malformed and there is nothing a consuming application can do to process this event, then the event gets discarded.
2. If the event fails to be processed but can be retried at a later time, the event is re-queued and processed at a later time most likely with some sort of exponential backoff algorithm in case of multiple retries.
3. If the consuming application is down, the event will not be delivered to it and will remain queued.

### Q. What happens if a consuming application fails to write to the event store after processing an event?
If the consuming application cannot write to the event store after processing an event, it would re-queue the event and process it at a later time (any actions that were completed in the previous run would be skipped).

### Q. Why write intermediary "in progress" events to the event store?
We introduce intermediary "in-progress" events as a mechanism to deal with any synchronous operations such as payment or shipping address updates or order cancellations that may occur concurrently. These synchronous operations depend on the order not being processed past certain states. We can illustrate how intermediary events help us with responding to these types of synchronous operations through a concrete example.

_Example_: Let's imagine a scenario where a customer initiates a shipping address update request. Now at the same time, shipping preparation is about to begin. The condition for the shipping address update to be successful is that shipping preparation should not be underway or complete. Now with the use of an "in-progress" event, we can update the event store which will ultimately be used to check whether the shipping address update should take place or not. If the "in-progress" event has been written to the event store, then we immediately know that the shipping address update should be disallowed. On the other hand, if the "in-progress" event has not been written to the event store, the shipping address update can be completed.

Now if we didn't use an "in-progress" event, there would still be ways to address this scenario however they could potentially introduce more complexity to the system. We could either determine whether shipping preparation is underway via the Order Shipping Service (potentially via an HTTP REST API), or we could allow the update to go through and publish an update event. The Order Shipping Service could then subscribe to this event and would then potentially need a compensating action to be performed if it had already undergone shipping preparation.

### Q. What happens if the Order Event Service writes to the event store but fails to publish the event?
If the Order Event Service successfully writes to the event store but fails to publish the event, then secondary backup mechanisms can be put into place to reattempt to publish at a later time, for example a batch job.

### Q. How are duplicate events handled?
Consuming applications will always record event ids in a local data store. It is important to distinguish between an event that was intentionally re-queued by a consuming application, and a "true" duplicate event. In the former case, re-queued events can be identified via message properties that indicate it is a re-queued event and thus will be processed by the consuming application. For the latter case, the event will get discarded if found in a consumer application's local data store but not identified as a re-queued event. 