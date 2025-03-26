# Readme: LemonSqueezy Integration

## Pocketbase Collections

This platform relies on a few Pocketbase collections to track payment and credit usage:

- checkout_intents
- lq_webhook_events
- lq_purchases
- lq_subscriptions
- credit_balances

lq stands for LemonSqueezy, i.e these collections are payment platform specific. In the future, if we work on integrating with Paddle for example, we need to create Paddle specific equivalents for these collections (Example: pd_webhook_events, pd_purchases etc). `checkout_intents` and `credit_balances` are intended to be used as payment platform agnostic collections. `checkout_intents` get populated as soon as a user clicks the `Buy` or `Generate Now` button. We create a checkout_intents record and pass it on to the payment platform along with the respective userId. All subsequent payment webhook payloads have both these fields for us to reconcile. Below are the details of these collections and their relations:

Github somehow does not like mermaid ER Diagrams. So, attached it as a png file (the empty box on the left is users collection)

![ER Diagram](./mermaid-diagram-2025-03-26-210356.svg)

## Payment

### One off purchase

The below collections are used to track one off purchases:

- checkout_intents
- lq_webhook_events
- lq_purchases
- credit_balances

```mermaid
sequenceDiagram
    actor User
    participant App as SolidStart App
    participant Payment as Payment Platform
    participant Queue as BullMQ Queue
    participant Worker as BullMQ Worker
    participant DB as PocketBase

    User->>App: Click Buy Button
    App->>DB: Create checkout_intent
    App->>Payment: Redirect to payment
    User->>Payment: Complete payment
    Payment->>App: Send order_created webhook
    App->>Queue: Queue webhook payload
    App->>Payment: Send 200 OK
    Queue->>Worker: Process order
    Worker->>DB: Update collections
    Note over Worker,DB: Update:<br/>- webhook_events<br/>- purchases<br/>- credit_balances
    Worker->>DB: Mark checkout_intent completed
```

Below is the detailed workflow, this is subjected to change over time. This detailed workflow should be used as a guide only. We should always check the source of truth, which is the code.

```mermaid
sequenceDiagram
    participant Worker as BullMQ Worker
    participant Lock as Redis Lock
    participant CI as Checkout Intent
    participant WE as Webhook Event
    participant User as User Service
    participant Purchase as Purchase Service
    participant Balance as Credit Balance
    
    Worker->>Lock: Acquire lock on checkout_intent_id
    
    Worker->>CI: Validate checkout_intent
    Note over Worker,CI: Verify intent exists<br/>and matches order
    
    Worker->>CI: Update with webhook_received & order_id
    
    Worker->>WE: Check for existing webhook_event
    alt Webhook already processed
        WE-->>Worker: Return success
    else No webhook event
        Worker->>WE: Create new webhook_event
        Note over Worker,WE: Status: pending<br/>Track processing steps
    end
    
    Worker->>User: Identify webhook user
    Note over Worker,User: Validate user_id<br/>from custom_data
    
    Worker->>Purchase: Validate variant & calculate credits
    Note over Worker,Purchase: Check plan validity<br/>Calculate total credits
    
    Worker->>Purchase: Create purchase record
    Note over Worker,Purchase: Store order details<br/>and credit amount
    
    Worker->>WE: Update processing status
    Note over Worker,WE: Mark purchase created
    
    Worker->>Balance: Add credits to user balance
    Note over Worker,Balance: Update total_credits
    
    Worker->>WE: Mark webhook completed
    Note over Worker,WE: Update processing status<br/>and mark completed
    
    Worker->>CI: Mark checkout completed
    Note over Worker,CI: Update status & timestamp
    
    Worker->>Lock: Release lock
```

### Subscription (New subscription only)

The below collections are used to track various stages of subscription:

- checkout_intents
- lq_webhook_events
- lq_purchases
- lq_subscriptions
- credit_balances

Unlike oneoff purchases, LemonSqueezy sends 4 webhook events, generally in the below order (the order is not guranteed) for a new subscription:

```mermaid
sequenceDiagram
    participant LS as LemonSqueezy
    participant App as Our Application
    
    Note over LS,App: New Subscription Purchase Flow
    
    LS->>App: 1. order_created
    Note right of App: Initial order details received
    
    LS->>App: 2. subscription_created
    Note right of App: Subscription record created
    
    LS->>App: 3. subscription_payment_success
    Note right of App: First payment confirmed
    
    LS--xApp: 4. license_key_created
    Note right of App: Not processed (unused)
    
    LS->>App: 5. subscription_updated
    Note right of App: Final subscription status updated
```

Each webhook event serves a specific purpose:

1. `order_created`: Contains the initial order information and payment details
2. `subscription_created`: Notifies that a subscription has been created for the order
3. `subscription_payment_success`: Confirms successful processing of the first payment
4. `license_key_created`: Notifies that a license key has been created (unused in this context)
5. `subscription_updated`: Updates the subscription with final status and details

Note: The `license_key_created` event is also sent by LemonSqueezy but is not processed by our application as we don't use license functionality for subscriptions.

The application enforces webhook processing order through strict validation checks and lock on checkout_intent_id:

```mermaid
stateDiagram-v2
    [*] --> PENDING: Checkout Intent Created
    PENDING --> ORDER_CREATED: subscription_order_created
    ORDER_CREATED --> SUBSCRIPTION_CREATED: subscription_created
    SUBSCRIPTION_CREATED --> PROCESSING: Updates Checkout Intent
    PROCESSING --> COMPLETED: subscription_payment_success
    COMPLETED --> UPDATED: subscription_updated
    
    state ORDER_CREATED {
        [*] --> ValidateCheckoutIntent
        ValidateCheckoutIntent --> ValidateNoPriorEvents
        ValidateNoPriorEvents --> CreateWebhookEvent
        CreateWebhookEvent --> [*]
    }

    state SUBSCRIPTION_CREATED {
        [*] --> ValidateOrderCreated
        ValidateOrderCreated --> ValidateOrderMatch
        ValidateOrderMatch --> CreateSubscription
        CreateSubscription --> [*]
    }

    state PROCESSING {
        [*] --> ValidateSubscriptionCreated
        ValidateSubscriptionCreated --> ValidateEventSequence
        ValidateEventSequence --> AllocateCredits
        AllocateCredits --> [*]
    }

    state UPDATED {
        [*] --> ValidatePaymentSuccess
        ValidatePaymentSuccess --> UpdateSubscriptionDetails
        UpdateSubscriptionDetails --> [*]
    }
```

```mermaid
sequenceDiagram
    participant CI as Checkout Intent
    participant WE as Webhook Events
    participant S as Subscription

    Note over CI,S: Validation Flow
    CI->>WE: 1. Validate Status
    WE->>WE: 2. Check Event History
    WE->>WE: 3. Verify Event Order
    WE->>WE: 4. Confirm Previous Complete
    WE->>S: 5. Process If Valid
```

Below are the detailed processing steps:

```mermaid
sequenceDiagram
    participant LS as LemonSqueezy
    participant App as SolidStart App
    participant Queue as BullMQ Queue
    participant Worker as BullMQ Worker
    participant Lock as Redis Lock
    participant CI as Checkout Intent
    participant WE as Webhook Events
    participant U as User
    participant P as Purchase
    participant S as Subscription
    participant CB as Credit Balance

    %% Initial Webhook Receipt
    LS->>App: subscription_order_created
    App->>Queue: Queue webhook payload
    App-->>LS: Return 200 OK immediately
    Queue->>Worker: Process order_created

    %% Order Created Processing
    Worker->>Lock: Acquire lock (checkout_intent_id)
    Worker->>CI: Validate (expect PENDING)
    Worker->>WE: Check No Prior Events
    Worker->>CI: Update with Order Details
    Worker->>WE: Create Event Record
    Worker->>U: Identify User
    Worker->>Worker: Validate Plan/Variant
    Worker->>WE: Mark Complete
    Worker->>Lock: Release lock

    %% Subscription Created Flow
    LS->>App: subscription_created
    App->>Queue: Queue webhook payload
    App-->>LS: Return 200 OK immediately
    Queue->>Worker: Process subscription_created
    Worker->>Lock: Acquire lock (checkout_intent_id)
    Worker->>CI: Validate (expect PENDING/PROCESSING)
    Worker->>WE: Verify order_created Complete
    Worker->>Worker: Validate Order & Variant Match
    Worker->>CI: Update to PROCESSING
    Worker->>WE: Create Event Record
    Worker->>U: Identify User
    Worker->>Worker: Calculate Credits
    Worker->>P: Create Purchase Record
    Worker->>S: Create Subscription Record
    Worker->>WE: Mark Complete
    Worker->>Lock: Release lock

    %% Payment Success Flow
    LS->>App: subscription_payment_success
    App->>Queue: Queue webhook payload
    App-->>LS: Return 200 OK immediately
    Queue->>Worker: Process payment_success
    Worker->>Lock: Acquire lock (checkout_intent_id)
    Worker->>Worker: Check Billing Reason (initial)
    Worker->>CI: Validate (expect PROCESSING)
    Worker->>WE: Verify Previous Events Complete
    Worker->>WE: Create Event Record
    Worker->>U: Identify User
    Worker->>S: Verify Subscription
    Worker->>CB: Add Credits to Balance
    Worker->>CI: Update to COMPLETED
    Worker->>WE: Mark Complete
    Worker->>Lock: Release lock

    %% Subscription Updated Flow
    LS->>App: subscription_updated
    App->>Queue: Queue webhook payload
    App-->>LS: Return 200 OK immediately
    Queue->>Worker: Process subscription_updated
    Worker->>Lock: Acquire lock (checkout_intent_id)
    Worker->>CI: Validate (expect COMPLETED)
    Worker->>WE: Verify All Events Complete
    Worker->>WE: Create Event Record
    Worker->>U: Identify User
    Worker->>S: Get Subscription
    Worker->>S: Update Details
    Note right of S: Update status, dates,<br/>card details, portal URLs
    Worker->>WE: Mark Complete
    Worker->>Lock: Release lock

    %% Error Handling Notes
    Note over Queue,Worker: Failed jobs are automatically<br/>retried by BullMQ
    Note over Worker,WE: Each step includes:<br/>1. Error handling<br/>2. Duplicate check<br/>3. Progress tracking

    %% State Transitions
    Note over CI: States: PENDING → PROCESSING → COMPLETED
    Note over WE: All events must be processed in sequence
    Note over S: Created → Active → Updated
```