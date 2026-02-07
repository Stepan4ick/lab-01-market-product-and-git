## Product Choice

- Wildberries;
- [Link to the product's website](https://www.wildberries.ru/);
- One of the biggest online supermarket in Russia. There are large range of absolutely different products.

## Main components

![Wildberries Component Diagram](../docs/diagrams/out/wildberries/architecture-component/Component%20Diagram.svg)

[Wildberries Component Diagram](../docs/diagrams/src/wildberries/architecture-component.puml)

- Auth & ID Service: component for authorization
- User Profile & Loyalty: component for managment of user's profile
- Notification Service: component for sending notifications and marketing offers
- Customer Website (SSR): component to manage the client website
- Customer Mobile App: component to access to the app

## Data flow

![Wildberries Sequence Diagram](../docs/diagrams/out/wildberries/architecture-sequence/Sequence%20Diagram.svg)

[Wildberries Sequence Diagram](../docs/diagrams/src/wildberries/architecture-sequence.puml)

This group of actions converts the user’s cart into an order, reserves the required items in stock, and starts the payment process.

First, the **Mobile App** sends a checkout request to the **Storefront Gateway** with the `cart_id` and selected `payment_method`.  
The Gateway forwards this request to the **Order Service (OMS)**, which creates a new order and stores it in the database with status `PENDING_PAYMENT`.

Next, the **Order Service** contacts the **Inventory Service** to reserve stock.  
It sends the list of `item_ids` and `quantities`.  
The Inventory Service updates the database by decreasing the available stock and confirms the reservation.

After the stock is reserved, the **Order Service** calls the **Payment Service** to initiate a transaction.  
The Payment Service communicates with the external **Bank/Acquirer API**, creates a payment session, and receives a `payment_url` (or token).

Finally, this payment URL is returned through the **Gateway** back to the **Mobile App**, and the user is redirected to complete the payment.

## Components and data exchange

- Mobile App → Gateway: `cart_id`, `payment_method`
- Gateway → Order Service: order creation request
- Order Service → Inventory Service: `item_ids`, `quantities`
- Inventory Service → Database: stock update
- Order Service → Payment Service: `order_id`, `amount`
- Payment Service → Bank API: payment session request
- Bank API → Payment Service: `payment_url`
- Gateway → Mobile App: `payment_url`

**Result:** the order is created, inventory is reserved, and the user proceeds to payment.

## Deployment

![Wildberries Deployment Diagram](../docs/diagrams/out/wildberries/architecture-deployment/Deployment%20Diagram.svg)

[Wildberries Deployment Diagram](../docs/diagrams/src/wildberries/architecture-deployment.puml)

The system is deployed as a distributed cloud-based architecture.

Users access the platform through web or mobile clients. Traffic first goes through external layers such as CDN, load balancers, and API gateways.

Core backend services (Cart, Order, Inventory, Payment, Notification, etc.) are deployed inside compute clusters (containers/VMs) in the cloud. These services communicate with each other over internal APIs.

Data is stored in dedicated storage clusters:

- relational databases for transactional data,
- cache (Redis) for fast access,
- message broker (Kafka) for asynchronous communication,
- object/blob storage for files and logs.

Background workers and analytics components run separately from the main services to process events and heavy jobs.

External systems such as payment providers, logistics, and partner services are integrated via public APIs outside the main infrastructure.

**In short:** clients → gateway → compute services → storage/messaging → external systems.

## Assumptions

- I assume the system follows a microservice architecture where Order, Cart, Inventory, Payment, and Notification services are deployed independently and communicate via internal APIs.
- I assume Kafka (or another message broker) is used for asynchronous communication between services, especially for order events and warehouse processing.
- I assume the Inventory service performs stock reservation during checkout to prevent overselling.
- I assume payment processing is delegated to external bank/acquirer providers and the system only manages payment sessions and callbacks.
- I assume Redis (or similar cache) is used to handle high traffic for carts, sessions, and frequently accessed product data.

## Open questions

- How is consistency between Order and Inventory guaranteed (transactions, saga pattern, or eventual consistency)?
- What scaling strategy is used during peak sales (auto-scaling, sharding, or traffic throttling)?
- How are payment failures and retries handled to avoid duplicate charges?
- What specific caching strategy is used for catalog and search (TTL, invalidation, or write-through)?
- How are warehouse and logistics services synchronized in real time with order status updates?
