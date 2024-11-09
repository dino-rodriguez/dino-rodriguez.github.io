---
layout: page
title: "Blockchain Notifications Product"
permalink: /posts/notifications-product
---

At Yoz Labs we developed a product that created and sent notifications based on blockchain activity. As one of the founding engineers I designed and built the backend system for this project.

![Notifications Product](/assets/notifications-product.svg)

Here’s a high-level overview of how the product works:
1. The user selects a verified smart contract on one of the supported blockchains and the system automatically fetches the ABI.
2. The user chooses the specific events or function calls to receive real-time notifications for.
3. The user creates a template for notifications using variables, functions, and text from the selected event or function, like this:
    {% raw %}
    ```
    User {{address}} sold {{weiToEth(value)}} ETH
    Transaction hash: {{txHash}}
    ```
    {% endraw %}
4. The user optionally sets up a filter to specify conditions for notifications. For example, the following filter only sends notifications for sales larger than 10 ETH:
    {% raw %}
    ```
    tradeType == "SELL" and weiToEth(value) > 10
    ```
    {% endraw %}
5. The user can preview recent notifications that would have been triggered by the function/event and filter combination to confirm their notifications look as expected.
6. Once satisfied, the user selects the delivery channel (e.g., Telegram, Discord, email) and enables notifications.

## Design of the System

![Notifications System Design](/assets/notifications-system-design.svg)

The system consists of several core components:
1. **Frontend**
    - Displays the web app for users to configure their notifications.
2. **Backend API**
    - Manages data for frontend interactions through a CRUD API.
3. **Producer Clusters**
    - Long-running processes that fetch blockchain data in real-time so that notifications messages can be generated.
4. **Message Constructor Cluster**
    - Processes raw data from the producers to construct notification messages for each relevant event or transaction.
5. **Sender Clusters**
    - Sends constructed notifications to recipients.
6. **Real Data Preview Service**
    - Loads raw blockchain data as parquet files allowing for fast, specific queries on events or functions that meet filter conditions.

Below we go into more detail on some of these components.

### Producer Cluster

Each blockchain had two producer processes: one for smart contract events and another for smart contract transactions (function calls). Our product monitored over six blockchains in real-time. The main loop for a producer worked as follows:
1. Retrieve the current block.
2. Fetch all transactions or events for the largest possible block range without exceeding limits, using exponential backoff if needed.
3. Send raw data for this block range to S3.
4. Filter the data for the contracts our users specified then enqueue raw data for each relevant event and transaction to be processed by the message constructors.

While producers primarily handled on-chain data we also explored additional data sources such as a producer fetching [Snapshot vote data.](https://snapshot.org/#/)

### Message Constructor Cluster

A pool of message constructor processes handled the following tasks:
1. Pulling raw events or transactions from a queue.
2. Decoding the blockchain data using the smart contract ABIs.
3. Retrieving relevant message templates for various user projects based on each event or transaction.
4. Evaluating boolean filter expressions to identify which message templates to keep.
5. Filling message templates with real values from the decoded data.
6. Identifying the delivery channel for each message and queuing it in the appropriate sender queue.

There were two main types of message constructors: one for events and another for transactions. Each type used a scalable pool of worker processes independent of any specific blockchain.

The decoding process was straightforward: using a contract ABI we could decode raw blockchain data into structured, human-readable information. Next we evaluate boolean filter expressions to determine which messages needed to be sent. Evaluating these expressions involved a custom templating language with several components:
1. **Plaintext**: Regular text.
2. **Variables**: Variables mapped directly to decoded data from the transaction or event which were declared using double curly braces.
3. **Functions**: Functions applied to data in the message template such as converting Wei to ETH (`weiToEth`) or fetching a token’s price (`fetchPrice`). Functions could modify both plaintext and variables.

Filters used this templating language but also allowed for complex, nested boolean logic (combinations with `not`, `and`, or `or`). An abstract syntax tree parsed and evaluated these boolean filter expressions.

After filtering, templates were evaluated by injecting decoded data and applying functions to construct messages. Any values fetched from third-party providers (like token price) were cached to reduce outbound requests and improve speed. Then constructed messages were queued on the appropriate persistent queue for delivery via the senders.

### Sender Cluster

For each delivery rail (Discord, Telegram, email), we ran a pool of sender processes. Each sender’s role was to take messages and deliver them to the correct recipient on the specified platform. The basic loop for a sender was as follows:
1. Perform validation checks such as ensuring the user hasn’t exceeded monthly limits and the message length is within limits.
2. Send the message via the specified delivery rail.
3. If sending fails re-queue the task up to `n` times, depending on the error code.
4. After a final success or failure log the outcome in the database to track all successful and failed messages.

### Backend API

The backend API primarily handled CRUD operations for the frontend allowing users to build and configure their projects. A notable feature was its ability to generate different types of message previews so users could see examples of the messages they’d receive. There were three types of message previews:
1. **Mock Data Filters**: Constructed mock data to render a sample message.
2. **Single TX Preview**: Accepted a transaction hash, fetched and decoded the transaction data, generated a message, and indicated whether any filters for that message were met.
3. **Real-Time Preview**: Reviewed the last `x` days of data for the transaction/event, applied filters and templating, and displayed the latest `n` messages that would have been generated.

### Real-Time Data Preview

We built the real-time data preview service to give users a view of what messages would have been sent recently without needing a specific transaction hash to test templates and filters.

This required access to historical blockchain data over a set time interval. Instead of loading this data into a traditional database—which would be costly and introduce complexity for executing functions within queries—we loaded the blockchain data as Parquet files on an EC2 instance. Parquet’s column-oriented format allows for fast file-based querying, and with DuckDB as our query engine we could execute functions directly within queries using [user-defined functions](https://duckdb.org/2023/07/07/python-udf.html).

To optimize query speed we partitioned the data as follows:
- **Top level**: Event or transaction.
- **Second level**: Contract address.
- **Third level**: Function method selector or event hash.
- **Fourth level**: Date-time buckets.

With this structure queries would target only the necessary data within specific time buckets, making retrieval extremely fast.

For applying filters in SQL-like DuckDB queries we converted each filter expression into an abstract syntax tree (AST) in Python. From this we could generate the corresponding `WHERE` clause. Filters with functions were executed as DuckDB UDFs within the query itself.

We exposed this real-time data preview service internally via an API to the backend API which in-turn proxied the functionality to the frontend.

## Final Thoughts

While this system was built specifically for the blockchain space its design could easily be adapted to create a general-purpose notifications platform. The modular structure —with producer clusters for data collection, message constructors for template-based notifications, and different senders — provides a solid foundation for building a notifications platform. I hope this post gave you some insight into building a system for real-time notifications. Thank you for reading!
