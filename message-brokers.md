# Message Broker Implementation

Current message broker: **RabbitMQ**

Current Protocol: **AMQP 0-9-1**

## Terminology

| Term | Meaning |
| ---------- | ----- |
| *Connection* | A physical TCP connection that needs to be established between a node and the broker itself. An AMQP connection is divided into a negotiated number of independent unidirectional ***channels***. |
| *Channel* | A unidirectional channel, and a subdivision of the TCP connection, which is used by the nodes (producers, consumers, queues) to communicate. | | *Message* | The data exchanged between the nodes.|
| *Message Broker* | A broker is simply the main server which implements the AMQP protocol core, and exposes an AMQP API, to which nodes can connect using a TCP connection. It is the part responsible for managing the queues, the exchange used for routing messages to queues, the storage and data persistance challenges, etc. It is the core part of the AMQP protocol. |
| *Consumer* | A node, which is an application server, that connects to the broker using a connection to subscribe (consume messages) from a node inside the broker (queue) when linked using a channel to that queue. |
| *Producer* | A node, which is an application server, that connects to the broker using a connection to produce (publish messages) to a node inside the broker (queue) when linked using a channel to that queue. |
| *Exchange* | Exchanges are AMQP 0-9-1 entities where messages are sent to. Exchanges take a message and route it into zero or more queues. The routing algorithm used depends on the exchange type and rules called bindings. |
| *Request Queue* | The main queue, from which any service expects to consume requests.<br><br> In other words, If a service needs to request any resources from a service initially, it produces a message on its ***request queue***. The service starts consuming|
| *Reply Queue* | The queue, from which a service expects a reply.<br><br> Whenever a ***producer*** publishes a message, while the [`waitForResponse`](#expected-parameters) option is true, the service expects to consume the reply message on its ***reply queue***. |
| *CorrelationId* | In distributed systems, each message between the different systems has an idempotent id which is a unique identifier created as soon as communication is initiated from the client, and it stays the same across the message's lifecycle. |
| *Request-Reply Pattern* | A message-exchange pattern, and a protocol, where a node sends a message to another node consuming that message, and it expects a reply from that node asynchronously. It either waits indefinitely until the node responds or until a **timeout** is reached, if defined. |
| *RPC<br>(Remote Procedure Call)* | A model used in the ***request-respond*** pattern, where the sender, or the requester, sends a message with the intent to execute a *procedure* remotely on the replier, or the receiver of the message, and expects to receive this *procedure*'s result as a response. |
| *functionName* | A field, which is specific to our implementation, that denotes the name of the procedure the ***producer*** wants to execute remotely on ***consumer***, then receive its result as a reply. We always send it in the [`headers`](#expected-parameters) parameter in the [`produce()`](#expected-parameters) method. |

## Implementation Components

| Class Name | Responsibilities |
| ---------- | ---------------- |
| `RabbitMQClient` | <ul><li>Establishes and manages a TCP [connection]((#terminology)) with the broker.</li><li>Creates 2 [channels]((#terminology)) with the broker (one for the producer, and one for the consumer).</li><li>Asserts the queues the ***Consumer*** expects to consume from in the broker.</li><li> Creates a ***Producer*** and a ***Consumer*** instances using composition.</li><li>Invokes the `consumeMessages()` method from the ***Consumer*** class to start subscribing on the established queues upon class instantiation.</li></ul>|
| `Producer` | <ul><li>Publishes messages on the establish producer channel injected into the class upon instantiation.</li></ul>|
| `Consumer` | <ul><li>Consumes messages from the [***reply queue***](#terminology).</li><li>Consumes messages from the [***request queue***](#terminology).</li><li>Instantiates the ***MessageHandler*** class and executes the `handleMessages()` method when a message is consumed from the [***request queue***](#terminology).</li><ul> |
| `MessageHandler` | <ul><li>Exposes the `handleMessages()` method, which might be named specific according to the service it's used in. <br><br>The `handleMessage()` method is called whenever a message is consumed, and executes the corresponding business logic handler method based on the `functionName` sent in the message's headers. Then, it produces the result to the [***reply queue***](#terminology) sent in the `replyTo` message property</li></ul> |

## `Produce()` method behavior

### Expected Parameters

**`data`**: The data expected to be consumed by the node you want to send a message to. Note that RabbitMQ only accepts the content data as a `Buffer` instance. Our implementation of the `produce()` method does that by default by calling `new Buffer(JSON.stringify(data))` internally.

**`queueName`**: The name of the target queue you want to send your message to, which is the same [***request queue***](#terminology) the node consumes data from.

**`correlationId`**: The id attached to the HTTP request that was sent initially from the client-side to the producer service.

**`waitForResponse`**: A boolean value which is the pillar of the **request-reply** pattern. If set to `true`, it means the producer would wait for a reply from the consumer, and would only return when a reply is received, or when a **timeout** is reached. Default value is `true`.

**`headers`**: An object that defines application-specific headers you want to be sent with your message. We use it to send metadata about the message in our implementation, mainly the [`functionName`](#terminology) field.

**`needToAssertQueue`**: A boolean that's when set to true tells RabbitMQ to make sure the queue exists before producing on it. Note that we cannot assert default queues made by RabbitMQ (they start with amq.*), which are [**reply queues**](#terminology) in our case. Default value is `false`ز

**`hasExpirationTime`**: A boolean that's when set to true tells RabbitMQ that it should time out when the expiration time period is reached. The time period is specific in the config file by the `messageExpiration` env variable. Default value is `true`.

## The Request-reply Pattern in Modern Message Brokers: A Distributed & Asynchronous Approach To Bridge The Execution Context Difference Between The Request and Its Reply

Request-reply is a message-exchange pattern, where a requestor sends a message to the replier, and expects a reply. It could be synchronous and blocking, or it could be asynchronous, where the requestor sends the message on a thread while it has a listener for the reply waiting on another thread. When a reply arrives the listener in the other thread is invoked and handles it using the callback you passed when you wrote the listener. Obviously, we implemented the asynchronous approach in our message broker implementation.

Let's take an example of the approach taken by one of the most popular message brokers, RabbitMQ, and how we could advance it to work in the use case of asynchronous communication between web services.

### How does RabbitMQ handle the Request-reply pattern by default?

Let's take the amqplib implementation for example. amqplib's implementation exposes a `sendToQueue()` method, which takes an `options` object as its 3rd parameter, and it accepts a `replyTo` field among others.

That field accepts a string that represents the queue name, from which the producer expects to consume the reply.

Let's dive into how the communication goes between the producer, the consumer, and the broker to better understand what's happening.

I am going to use a code example in *Express.js*, while using the *amqplib* client library to connect to a running Rabbitmq broker, just for the sake of clarity, but the concepts can be appliesd interchangeably in any other framework or language.

Let's start!

1. The client sends a regular HTTP request to **service A**, which is an application server that exposes a REST API for clients to request resources from. That request has a `correlationId`, which is a unique identifier for it across the different services.
2. In order for **service A** to get all the resources the client requested, it needs to talk to **service B** (act a producer), execute some logic over there, and get the result. Note that **Service A**, and **service B** are connected to a message broker using a TCP connection.
3. **Service B** is registered at the broker to be consuming messages from **`RequestQueueB`**
4. For **service A** to dispatch a message to **service B**, it needs to send a message to the broker at **`RequestQueueB`**.

        function getResourcesFromServiceB(messageContent, queueName, options) {
            /**
             * Assuming you established connection using amqplib to a rabbitmq broker,  
             *  then stored it in the amqplibConnectionChannel variable used below.
             * The channel interface exposes the sendToQueue method we talked about.
             **/
            return amqplibConnectionChannel.sendToQueue(
                queueName,
                Buffer.from(messageContent), 
                options
            );
        }
        
        app.on("/resources", async (request, response) => {
            const resourcesFromServiceB = await getResourcesFromServiceB(
                request.body,
                "RequestQueueB",
                {
                    correlationId: "clientRequestId1"
                }
            );

            response.json(resourcesFromServiceB);
        });
5. The broker then notifies the service consuming from **`RequestQueueB`** (**service B**).
6. Then **service B** receives the message from the broker, executes the requested procedure, and now it's ready to reply to **service A**. Now what? This is **service B** now:

        /**
         * The channel interface in amqplib exposes a method called consume,
         * which is used to set up a listener that consumes messages from a queue.
         * It also accepts a callback handler to call when a message is received.
         * For now, this handler will hold a string and do nothing untill we proceed.
         */
        amqplibConnectionChannel.consume("RequestQueueB", (receivedMessage) => {
            const resourcesThatServiceANeeds = "Username";
            // What to do now?
        });

7. In order for **service A** to be able to receive a reply from **service B**, it needs to act as consumer as well. In other words, it needs to be consuming from some other queue, where it can receive replies from the services it talks to.

        amqplibConnectionChannel.consume("ReplyQueueA", (receivedReply) => {
            /**
             * Wait, how to access this receivedReply in the other context in the HTTP method  
             * handler below to send it back to the client?
             */
        });

        function getResourcesFromServiceB(messageContent, queueName, options = {}) {
            /**
             * Assuming you established connection using amqplib to a rabbitmq broker,  
             *  then stored it in the amqplibConnectionChannel variable used below.
             * The channel interface exposes the sendToQueue method we talked about.
             **/
            return amqplibConnectionChannel.sendToQueue(
                queueName,
                Buffer.from(messageContent), 
                options
            );
        }
        
        app.on("/resources", async (request, response) => {
            const resourcesFromServiceB = await getResourcesFromServiceB(
                request.body,
                "RequestQueueB",
                {
                    correlationId: "clientRequestId1"
                }
            );

            response.json(resourcesFromServiceB);
        });
8. Based on what we said, **service A** is now consuming from **`ReplyQueueA`**. How will **service B** know that it can reply to **service A** on **`ReplyQueueA`**? Well, **service A** must have sent a property in its produced message which includes the name of the queue it expects a reply on, which is **`ReplyQueueA`** in **service A**'s case. That property is called `replyTo` in amqplib, and it's sent in the options object which accompanies the produced message.

        const resourcesFromServiceB = await getResourcesFromServiceB(
                request.body,
                "RequestQueueB",
                {
                    correlationId: "clientRequestId1",
                    replyTo: "ReplyQueueA"
                }
        );
9. So now **service B**, after finishing all the logic it needed to handle when it consumed the first message on its queue (**`RequestQueueB`**), proceeds to check the `replyTo` field in the message.
10. Then **service B** starts to act as a producer, and sends to the broker that it needs to produce a message to **`ReplyQueueA`**, which is in the `replyTo` property of the message.

        amqplibConnectionChannel.consume("RequestQueueB", (receivedMessage) => {
            const resourcesThatServiceANeeds = "Username";
            // What to do now?
            amqplibConnectionChannel.sendToQueue(
                receivedMessage.properties.replyTo,
                Buffer.from(resourcesThatServiceANeeds);
                {
                    correlationId: receivedMessage.properties.correlationId
                }
            );
        });
11. The broker then notifies the service consuming from **`ReplyQueueA`** (**serviceA**)
12. Then service A receives the message from the broker, and proceeds to respond to the client. But wait, where is the client?

Remember when **service A** first produced to **service B**? That actually ended with an acknowledgement from the broker that the message was put on **`RequestQueueB`**, and that was it for this execution context.

Then when **service A** consumed the reply message from **service B**, that was a totally different execution context inside the consuming method used when you set up a listener on this queue.

The question is:

How to bridge the gap between those two different contexts to receive the message from **service B**, while you still have access to the keep-alive HTTP connection between **service A** and the client, so that you can respond to the client from **service A** with the content you received as a reply from **service B**?

In an event-driven environment such as Node.js, The `events` module along with promises come to the rescue!

Going back to **service A**, we need to add three final things to bridge the gap.
First, when we produce the message that came from the client we need to set an event listener on that client's request `correlationId`, because it's a unique identifier.

Then, to return the value received when that listener catches an event emitted with the same `correlationId`, we need to wrap it in a promise which we can wait on in the client's context.

Finally, on the consumer that consumes from **ReplyQueueA**, we need to fire an event whenever a message arrives as a reply with the message's `correlationId`. That event's value would be the received reply's content. Therefore, the client handler, which was waiting on the promise returned from the `getResourcesFromService` method to resolve, will have access to the data from **service B** and can send it back to the client using `res.json`.

        const EventEmitter = require("events");
        cosnt eventHandler = new EventEmitter();
    
        amqplibConnectionChannel.consume("ReplyQueueA", (receivedReplyMsg) => {
            /**
             * Now, this consumer will fire an event named by the correlationId it receives
             */
             eventHandler.emit(
                receivedReplyMsg.properties.correlationId,
                receivedReplyMsg.content
             );
        });

        function getResourcesFromServiceB(messageContent, queueName, options = {}) {
            amqplibConnectionChannel.sendToQueue(
                queueName,
                Buffer.from(messageContent), 
                options
            );
            /**
             * Now, we'll return a promise, 
             * which is going to resolve with the received value from the listener
             */
             return new Promise((resolve, reject) => {
                eventHandler.once(options.correlationId, (dataSentFromEmitter) => {
                    resolve(dataSentFromEmitter);
                    /**
                     * Of course, you'd be doing error handling here,
                     * where you can use reject to return errors inside
                     */
                });
             })
        }
        
        app.on("/resources", async (request, response) => {
            const resourcesFromServiceB = await getResourcesFromServiceB(
                request.body,
                "RequestQueueB",
                {
                    correlationId: "clientRequestId1"
                }
            );
            /*
             * Now the resourcesFromServiceB variable holds the "Username" string,
             * which was sent * from service B
             */ 
            response.json(resourcesFromServiceB);
        });

<!-- Talk about acknowledgements -->
<!-- Talk about default exchange and unique reply queue -->
<!-- Talk about durable and autoDelete -->
