# OrderManagement

Client (Spring Boot) – uses WebClient

API Gateway – JWT, routing, resilience

Order Management Service

Inventory Service – Feign client for checking stock

MongoDB – Orders & Customers

           [ Client Spring Boot App ]
                       |
                       |  REST (WebClient) with JWT
                       V
             [ API Gateway (JWT, CB) ]
                       |
                       | Routes to services
                       V
          [ Order Management Service (MVC) ]
                       |
                       | Feign (REST)
                       V
            [ Inventory Service (MVC) ]
                       |
                       V
                 [ Inventory DB ]

                       |
                       V
               [ Orders & Customers DB ]

Component 1 — Client Spring Boot Application
Purpose

Simulates an external application placing orders.

Responsibilities

Accept user input (via REST endpoint)

Build OrderRequest

Add JWT token

Call API Gateway using WebClient

Get acknowledgment response

Internal Structure
client/
 ├── controller/
 │     └── ClientController.java
 ├── service/
 │     └── OrderClientService.java
 ├── config/
 │     └── WebClientConfig.java
 ├── dto/
 │     ├── OrderRequest.java
 │     ├── OrderResponse.java
 │     └── ErrorResponse.java
 └── ClientApplication.java

Client Flow
User → REST → ClientController → OrderClientService → WebClient → API Gateway

🔹 Component 2 — API Gateway (Spring Cloud Gateway)
Responsibilities

Validate JWT token

Route requests

Add rate limiting, circuit breaker, fallback (Resilience4j)

Logging

Routes
/orders/**    → order-service
/inventory/** → inventory-service   (optional external access)

Why Reactive Gateway?

Because Spring Cloud Gateway always uses WebFlux internally, but backend services can be MVC.

🔹 Component 3 — Order Management Service (Spring MVC)
Responsibilities

Receive order request (from Gateway)

Validate fields

Set PENDING status

Call Inventory Service via Feign

Based on response:

CONFIRM order

or FAIL with OUT_OF_STOCK

Persist order & customer

Return acknowledgment to client

Order Flow
API Gateway
    |
    V
Order Controller
    |
    V
OrderService (business logic)
    |
    V
FeignClient → Inventory Service
    |
    V
Save Order in MongoDB
    |
    V
Return OrderResponse

Order Status Flow
PENDING → INVENTORY_CHECKING → CONFIRMED
                 |                
                 └──> FAILED_OUT_OF_STOCK

Collections

Orders

orderId
customerId
itemId
quantity
price
totalCost
status
timestamps


Customers

customerId
name
phone
email
address

🔹 Component 4 — Inventory Service
Responsibilities

Check if item is in stock

Return simple payload:

{
  "itemId": "ITEM10",
  "available": true,
  "availableQuantity": 45
}

Tech

Spring Boot MVC

Optional MySQL/Postgres/MongoDB

Endpoints
GET /inventory/check/{itemId}?qty=10

Internal Flow
Controller → InventoryService → DB → Response

Response

AVAILABLE

OUT_OF_STOCK

Order Service Uses FeignClient
@FeignClient(name="inventory-service")
public interface InventoryClient {
    @GetMapping("/inventory/check/{itemId}")
    InventoryResponse check(@PathVariable String itemId, @RequestParam int qty);
}

🔹 5 — MongoDB (Orders + Customers)

Orders stored with MOST RECENT status

You can add OrderStatusHistory later if needed

📌 3. Sequence Diagram (Full System)
Client App
   |
   | POST /client/place-order
   |
   V
API Gateway
   |
   | POST /orders
   |
   V
Order Service
   |
   | Feign → /inventory/check
   |
   V
Inventory Service
   |
   | STOCK_AVAILABLE / OUT_OF_STOCK
   |
   V
Order Service
   |
   | Save customer, save order
   |
   V
API Gateway
   |
   | Response: orderId + status
   |
   V
Client App
