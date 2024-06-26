# Web-scale Data Management Project Team 9


NOTE TO REVIEWERS


The transactions that are processed by RabbitMQ are demonstrably eventually consistent (if executed on 1 queue). However, with the current implementation, the [benchmark](https://github.com/delftdata/wdm-project-benchmark) that is provided will not agree as the evaluation starts before RabbitMQ can finish processing all messages.


Inserting a sleep statement of ~10 seconds before the evaluation step will ensure the database ends up being consistent. The logs do not give proper results based on our current implementation.


### Running the code
- Copy `.env.example` and rename it to `.env`. Change environment to your liking.
- Run ```python generate_compose.py``` to create the docker-compose file based on the amount of consumers needed, which can be defined in ```.env```.
- To then start the cluster, you can run: ```docker-compose -f docker-compose.yml -f consumer-compose.yml up```.


### Design decisions
In this README we give an overview of our biggest design decision with more elaborate explanations in the [wiki](https://github.com/mennohie/wdm24-team9/wiki).
- We chose to use RabbitMQ as our message broker [Message broker](https://github.com/mennohie/wdm24-team9/wiki/Concept-Message-Broker). Due to using this message broker, we make the system more scalable using asynchronous communication.
- The consumers in this system can be scaled horizontally. This means that we can add more consumers to the system to handle more messages. This is done by running multiple instances of the consumer.
- We use [SAGAs](https://github.com/mennohie/wdm24-team9/wiki/Concept-SAGAS) to handle the transactions in the system. This allows us to combine the information from multiple services (payment and stock) to create a consistent state in the system. Publisher try to process the order step by step, if one step fails the publisher will send a message to rollback the previous steps.
#### Request Status
Our design has the Order Service directly answer requests to `/orders/addItem/{order_id}/{item_id}/{quantity}` and `/orders/checkout/{order_id}`, reporting that the message has been correctly send from the order service.
The client or user has this response relayed back to them with an added request `correlation_id` that corresponds to that specific request. The client can then poll the order service for a request status via `/orders/status/{correlation_id}`.
The Message consumer updates this request status according to whether the request has been handled successfully or failed somewhere along the road.
#### Sharding
We have chosen to shard the checkout queues based on user ID, this means that all calls from a certain user get assigned to the same queue. This assures that the order of the call is maintained. For example, if a user tries to buy a product and then tries to buy another product, the order of these calls is maintained.
#### Database Locking
Due to our asynchronous communication, the system can become inconsistent if multiple users are trying to buy the same product at the same time. In the method `subtract_stock` of the Stock Service, the new stock is calculated in memory and then updated to the database. If this calculation happens with a value that is outdated, this will create an inconsistency. Therefore, we have implemented a database lock for this single operation. There seems to be little reason to implement a lock for any other operation, so the database will not be locked for any other operation to prevent unnecessary bottlenecks. The implementation of the lock does not seem to impair (multi-thread) performance too much.
#### Fault Tolerance
For Fault Tolerance we use persistent queues, messages and dbs to ensure that messages are not lost. If a consumer fails, the message will be requeued and be processed once the queue is available again. This ensures that no messages are lost in the system.
- If a service dies upon startup it will automatically reconnect to the system.
- If a consumer processing the queue dies it will continue where it left off once it is available and connected again.
