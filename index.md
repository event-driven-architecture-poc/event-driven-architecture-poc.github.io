# Project
This project demonstrates the use of an event-driven microservice architecture using event sourcing. The architecture will be applied to a simplified e-commerce website with functionality outlined below. This project is primarily implemented as a passion project and for learning purposes. 

## System Overview
The e-commerce website will allow customers to place orders, view the status of their orders, update payment and shipping details for their order, and cancel their order.

### Order Lifecycle
The lifecycle for a customer's order is as follows:
1. A customer places an order through the website. Once the order has been created, the customer receives an order confirmation email notification.
2. An attempt is made to authorize the customer's payment. If the authorization fails, the customer receives an email notification indicating that the payment authorization failed.
3. Once the payment is authorized successfully, the order is prepared for shipping.
4. Once the order is shipped out for delivery, the customer will receive an email notification indicating that the order has shipped out for delivery.

### Customer Actions
Customers will also be allowed to do the following:
- View the current status of their order
- Update their payment method for the order, given that the existing payment authorization is not in progress or complete.
- Update their shipping address for the order, given that shipping preparation is not in progress or complete.
- Cancel their order, given that shipping preparation is not in progress or complete. If their order is cancelled, the customer will receive an order cancelled email notification.