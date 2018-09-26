# RabbitMQ Tracing Guide


Trace messages as they traverse RabbitMQ

I want to know :  
a) which messages are published through exchange X  
b) which messages are being delivered by queue Y

We can use [Firehose](#Firehose) or [rabbitmq_tracing plugin](rabbitmq_tracing-plugin) to get to know a) and b).  
We can use [Rabtap](#Rabtap) to get to know a).

**TL;DR** None of the tools mentioned above will tell us which user published or received the message and/or from which connection (ip, port). The connection and user information provided by the **rabbitmq_tracing plugin** is relative to the user used by the plugin, it is not the user that either received or published the message.

## Firehose

The **Firehose** allows us to trace messages as they traverse RabbitMQ and sends a copy of each traced message to an exchange. It is up to us to bind a queue to that exchange in order to get those messages.  

### Who can use it

Only **Operators** -those who have access to **RabbitMQ** via `rabbitmqctl`- can enable this feature. However, once it is enabled, any user which has permission to bind a queue to the `amq.rabbitmq.trace` exchange can use this feature to trace messages.

### How to enable/configure it

The **Operator** runs `rabbitmqctl trace_on` to **enable** tracing **ONLY** on the *default vhost* and on the node from where we are running `rabbitmqctl trace_on`. However, it can also be enabled on other *vhosts* by passing the argument `-p <vhostName>` and/or on other nodes by passing `-n <nodeName>`.

**TL;DR** It is really important to understand that **Firehose** will **ONLY** trace messages that were published through the node(s) we are tracing on and likewise it will **ONLY** trace messages that were delivered to a client via the node(s) we are tracing on.

### How to start tracing

In the previous section we enabled tracing but we are not actually tracing any messages yet. To actually trace messages a RabbitMQ user needs to *bind* a queue (this is the destination of the traced messages) to the `amq.rabbitmq.trace` exchange and use the appropriate *routing key* based on what we want to trace:
  - `#` trace every message sent to any exchange and delivered by any queue
  - `publish.#` trace every message sent to any exchange
  - `deliver.#` trace every message delivered by any queue
  - `publish.X` trace every message sent to the exchange `X`

### How to stop tracing

To stop tracing, remove any bindings to the `amq.rabbitmq.trace` exchange.

To disable tracing entirely, the **Operator** needs to run `rabbitmqctl trace_off`.

**TL;DR** If [rabbitmq_tracing plugin](#rabbitmq_tracing-plugin) is also enabled and the **administrator** stops the last **Trace**, it will disable **Firehose** tracing entirely in addition to stopping the last **Trace** instance.

### Implications of tracing messages with Firehose

Having **Firehose** enabled (i.e. we have run `rabbitmqctl trace_on`) has a negative performance impact even when there are no bindings on `amq.rabbitmq.trace`. Furthermore, it has a significant performance penalty when we define traces.

On an experiment run to asses the performance impact of **Firehose** this is what we found out:
- We use RabbitMQ 3.7.6 running with Erlang 20.3.8.1 on a MBP (2,5 GHz Intel Core i7) and `rabbitmq-perf-test-1.1.0`
- `bin/runjava com.rabbitmq.perf.PerfTest -u test` could produce on average 24k msg/sec
- After running `rabbimtqctl trace_on`, the message throughput dropped to 19k msg/sec
- After binding a queue to `amq.rabbitmq.trace` with the routing key `#`, the message throughput dropped to 9k msg/second
- After running `rabbimtqctl trace_off`, the message throughput increased again to 24k msg/sec. It does not matter whether we have any queue bound to `amq.rabbitmq.trace`.

## rabbitmq_tracing plugin

The **rabbitmq_tracing plugin** allows us to trace messages as they traverse RabbitMQ and sends them to a log file that we can later on view -if we have access to the node- or we can download it via the Management UI. It uses the [Firehose](#Firehose)

### Who can use it

Only **Operators** -those who have access to **RabbitMQ** via `rabbitmqctl`- can enable this feature. And only **administrators** -those users with **administrator** *user tag*- can use this feature to trace messages. No other *user tags* grants access to this feature.

### How to enable/configure it

We need to carry out 2 steps:
  1. First, the **Operator** has to enable the plugin

```
rabbitmq-plugins enable rabbitmq_tracing
```
  2. Second, the **administrator** has to configure in `rabbitmq.config`, the location where to store the trace log files. It also has to configure the RabbitMQ credentials the plugin uses to create a queue, bind it to the `amq.rabbitmq.trace` and consume messages from the queue to dump them onto a log file.

  ```
  ....
  {rabbitmq_tracing,
     [
         {directory, "/var/vcap/sys/log/rabbitmq-server/tracing"},
         {username, <<"admin">>},
         {password, <<"password">>}
     ]
  },
  ....
  ```
  3. **Operator** has to restart RabbitMQ cluster to take the configuration changes
  4. The **administrator** should see a new option called **Tracing** in the the **Admin** tab in RabbitMQ Management UI

### How to start tracing

1. Go to the Management UI > *Admin* tab > *Tracing* menu.
2. Choose the node you want to trace on
  ![Choose node](rabbitmq_tracing_choose_node.png)
3. Give it a name to your trace. Choose the vhost. We can limit the amount of bytes of the payload we want to log.  
  **Warning**: We should delete the trace file if it exists and it is not empty otherwise the plugin will fail to create the trace
4. Choose which messages we want to trace:
  - `#` trace every message sent to any exchange and delivered by any queue
  - `publish.#` trace every message sent to any exchange
  - `deliver.#` trace every message delivered by any queue
  - `publish.X` trace every message sent to the exchange `X`

After we add the trace we can see:

- the trace
  ![trace](rabbitmq-tracing-trace.png)
- one extra queue
  ![trace](rabbitmq-tracing-queues.png)
- one extra connection (and a consumer channel). See the username matches with user configured in the plugin, i.e. `admin`.
  ![trace](rabbitmq-tracing-conn.png)


**TL;DR** It is really important to understand that `rabbitmq_tracing` will **ONLY** trace messages that were published via the node(s) we are tracing on and likewise it will **ONLY** trace messages that were delivered via the node(s) we are tracing on. See section [Tracing in action](#Tracing-in-action) for further details.

**TL;DR** Purged messages are not delivered hence they are not traced at all.

**TL;DR** Binding our own queues directly to the `amq.rabbitmq.trace` will not work when we are using the plugin. It only works when we have **Firehose** tracing on.

**TL;DR** If we have **Firehose** enabled and also a **Trace**, each one will work as expected. The **Trace** will trace the event in the log and the **Firehose** will send the message to any bound queue to the `amq.rabbitmq.trace` exchange. The **Trace** will only receive the event once though.  

Is it possible to capture publishing via the default exchange ?

### How to stop tracing

To stop a tracing, simply click on the `Stop` button on the trace we want to stop. It will close the consumer connection and delete the internal queue. If the trace generated a trace log file, it will stay until we delete it by clicking on the `Delete` button.

**Do not delete the file before stopping the trace**. The trace will continue tracing messages but they will not be logged. So effectively we have stopped tracing.

To disable tracing entirely, the **Operator** needs to disable the plugin by running `rabbimq-plugins disable rabbitmq_tracing`

### How to view traced messages

To see the captured messages we have 3 options:
- Download them via the Management UI by clicking on the trace log,
- Download them via the Management API: `GET    /api/trace-files/<name>`
- View the file directly from the RabbitMQ's node

### Tracing in action
Let's use the **rabbitmq_tracing plugin** to trace publishing and delivery of messages in a 2 node (`rmq/0` and `rmq/1`) RabbitMQ cluster.

**Tracing message publishing**

We will see 3 scenarios where we demonstrate that in order to trace every published message, we need to have a trace on all the nodes the AMQP clients are publishing to.

Scenario 1:
- Define a trace on `rmq/0` node (`10.0.1.0`) with pattern `publish.amq.direct` and name `publish.amq.direct`
- Send message via `rmq/0` node to `q-rmq0` through `amq.direct` exchange

```
  [AMQP Client]--publish(amq.direct#q-rmq0)--->[ ** RMQ Node rmq/0 ** ]----->{ q-rmq0 }

                                               [ RMQ Node rmq/1       ]------{ q-rmq1 }
```
Run this script from `rmq/0` node:
```
rabbitmqadmin publish routing_key=q-rmq0 exchange=amq.direct payload="message 1"
```

- Outcome: **message is logged !!!**
```

```

Scenario 2:
- Same as in scenario 1; define a trace on `rmq/0` node with pattern `publish.amq.direct`
- Send message via `rmq/1` node (`10.0.1.1`) to `q-rmq1` through `amq.direct` exchange

```
                                        [ ** RMQ Node rmq/0 ** ]-----{ q-rmq0 }

  [AMQP Client]--publish(amq.direct)--->[ RMQ Node rmq/1       ]---->{ q-rmq1 }

```
Run this command from `rmq/1` node:
```
rabbitmqadmin publish routing_key=q-rmq1 exchange=amq.direct payload="publish - scenario 1"   
```
- Outcome: **Nothing gets logged !!!**


Scenario 3:
- Same as in scenario 1, i.e. define a trace on `rmq/0` node with pattern `publish.amq.direct`
```
                                        [ ** RMQ Node rmq/0 ** ]--->{ q-rmq0 }

  [AMQP Client]--publish(amq.direct)--->[ RMQ Node rmq/1       ]----{ q-rmq1 }

```
- Send message via `rmq/1` node to `q-rmq0` through `amq.direct` exchange
  Run this command from `rmq/1` node:
```
   rabbitmqadmin publish routing_key=q-rmq0 exchange=amq.direct payload="publish - scenario 2"   

```
- Outcome: **Nothing gets logged !!!**

**Tracing message delivery**

Scenario 1:
- Define a trace on `rmq/0` node with pattern `deliver.q-rmq0`
- Consume message via `rmq/0` node from `q-rmq0`
```
     [ ** RMQ Node rmq/0 ** ]---{ q-rmq0 }------------->[AMQP Client]

     [ ** RMQ Node rmq/1 ** ]
```
  Run this command from `rmq/0`:
```
  rabbitmqadmin publish routing_key=q-rmq0 exchange=amq.direct payload="delivery - scenario 1"   
  rabbitmqadmin get queue=q-rmq0 count=1 ackmode=ack_requeue_false
```
- Outcome: **Message gets logged on rmq/0 !!!**
```
================================================================================
2018-09-26 14:22:36:558: Message received

Node:         rabbit@rmq0-rmq-mrosales-20180925
Connection:   <rabbit@rmq0-rmq-mrosales-20180925.1.18253.0>
Virtual host: /
User:         admin
Channel:      1
Exchange:     amq.direct
Routing keys: [<<"q-rmq0">>]
Queue:        q-rmq0
Properties:   []
Payload:
delivery - scenario 1
```

Scenario 2:
- Same as scenario 1, define a trace on `rmq/0` node with pattern `deliver.q-rmq0`
- Consume message via `rmq/1` node from `q-rmq0`
```
      [ ** RMQ Node rmq/0 ** ]
         ( q-rmq0 )                                                  
              |
              \/
      [ RMQ Node rmq/1       ]-------------->[AMQP Client]

```
  Run this command from `rmq/1`:
```
rabbitmqadmin publish routing_key=q-rmq0 exchange=amq.direct payload="delivery - scenario 2"   
rabbitmqadmin get queue=q-rmq0 count=1  
```
- Outcome: **Message is not logged because it is delivered thru a node which has no tracing**

Scenario 3:
- Define trace on both `rmq/0` and `rmq/1` with pattern `deliver.q-rmq0`
- Consume message from `q-rmq0` via `rmq/1` node
```
      [ ** RMQ Node rmq/0 ** ]
         ( q-rmq0 )                                                  
              |
              \/
      [ RMQ Node rmq/1       ]-------------->[AMQP Client]

```
Run this command from `rmq/1`:
```
rabbitmqadmin publish routing_key=q-rmq0 exchange=amq.direct payload="delivery - scenario 3"   
rabbitmqadmin get queue=q-rmq0 count=1  
```
- Outcome: **Message is logged in both nodes !!!!**


**Conclusion**: If we don't know where (i.e. RabbitMQ node) messages are being published from or delivered to, it is best to add a **Trace** to each node of the cluster. Unless we have very specific cases like these ones:
  - publisher application is currently connected to `rmq/0` node and we want to capture what message is publishing, or
  - consumer application is currently connected to `rmq/1` node and we want to capture what messages is receiving


### Implications of tracing messages with rabbitmq_tracing plugin

Having `rabbitmq_tracing` plugin enabled has no negative performance impact if we have not defined any traces yet.

On an experiment run to asses the performance impact of **rabbitmq_tracing** plugin this is what we found out:
- We use RabbitMQ 3.7.6 running with Erlang 20.3.8.1 on a MBP (2,5 GHz Intel Core i7) and `rabbitmq-perf-test-1.1.0`
- **rabbitmq_tracing** was enabled
- `bin/runjava com.rabbitmq.perf.PerfTest -u test` could produce on average 24k msg/sec
- Define a new trace via the Management UI. `Name = "tracer-1", Pattern = "#"`
- Message throughput dropped to again to 8 msg/sec

We observe a slightly worse performance compared to using **Firehose**.


### Rabtap

[Rabtap](https://github.com/jandelgado/rabtap) taps messages being sent to exchanges using RabbitMQ exchange-to-exchange bindings without affecting actual message delivery.

As opposed to the previous two options -**Firehose** and **Tracing plugin**, it does not capture when messages are delivered from queues.

Traced messages are either printed directly onto the terminal where we running **RabTap** or onto a file.

The command below traces all messages sent to the `amq.fanout` exchange and prints the messages on the terminal.
```
rabtap tap --uri amqp://guest:guest@localhost/ amq.fanout:
```


## Tracing AMQP traffic


### Tracer

```
  [AMQP client]<---->[Tracer]<---->[RabbitMQ server]
```
TODO

### TCPDUMP + Wireshark

TODO
