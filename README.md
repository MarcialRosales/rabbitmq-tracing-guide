# RabbitMQ Sniffer Guide

- Sniff messages vs AMQP traffic between client-brokers
- Production usage vs Development usage ; performance impact


## Sniff messages as they traverse RabbitMQ

I want to know :  
a) which messages are flowing through exchange X  
b) which messages are being delivered by queue Y

We can use [Firehose](#Firehose) or [rabbitmq_tracing plugin](rabbitmq_tracing-plugin) to get to know a) and b).  
We can use [Rabtap](#Rabtap) to get to know a).


### Firehose

Only **Operators** -those who have access to **RabbitMQ** via `rabbitmqctl`- can enable this feature.

Run `rabbitmqctl trace_on` to **enable** tracing on the *default vhost*. It can also be enabled on other *vhosts* by passing the argument `-p <vhostName>`.

So far we have enabled tracing but we are not actually sniffing any messages yet. To actually sniff messages we need to *bind* a queue (this is the destination of the sniffed messages) to the `amq.rabbitmq.trace` exchange and use the appropriate *routing key* based on what we want to sniff:
  - `#` sniff every message sent to any exchange and delivered by any queue
  - `publish.#` sniff every message sent to any exchange
  - `deliver.#` sniff every message delivered by any queue
  - `publish.X` sniff every message sent to the exchange `X`

Having *Firehose** enabled (i.e. we have run `rabbitmqctl trace_on`) has a negative performance impact even when there are no bindings on `amq.rabbitmq.trace`. And it has a significant performance penalty when we define traces.

On an experiment run to asses the performance impact of **Firehose** this is what we found out:
- We use RabbitMQ 3.7.6 running with Erlang 20.3.8.1 on a MBP (2,5 GHz Intel Core i7) and `rabbitmq-perf-test-1.1.0`
- `bin/runjava com.rabbitmq.perf.PerfTest -u test` could produce on average 24k msg/sec
- After running `rabbimtqctl trace_on`, the message throughput dropped to 19k msg/sec
- After binding a queue to `amq.rabbitmq.trace` with the routing key `#`, the message throughput dropped to 9k msg/second
- After running `rabbimtqctl trace_off`, the message throughput increased again to 24k msg/sec. It does not matter whether we have any queue bound to `amq.rabbitmq.trace`.


### rabbitmq_tracing plugin

Only **administrators** -those users with **administrator** *user tag* can define their own traces provided the `rabbitmq_tracing` plugin is enabled. No other *user tag* grants access to this feature. This feature is available via the rabbitmq management console.

Having `rabbitmq_tracing` plugin enabled has no negative performance impact when no traces have been defined yet. But it has a slightly worse performance than using **Firehose**.

To actually sniff messages we need to add a trace via the management console. Messages are written to a log file in either text or json format. Enter onto the `pattern` field what we want to sniff:
  - `#` sniff every message sent to any exchange and delivered by any queue
  - `publish.#` sniff every message sent to any exchange
  - `deliver.#` sniff every message delivered by any queue
  - `publish.X` sniff every message sent to the exchange `X`

To see the captured messages we have 2 options:
- Download a log file with all the messages via the management console, or
- View the file directly from the RabbitMQ's node
The former option can be a bit problematic if the log file is too large to download.

On an experiment run to asses the performance impact of **rabbitmq_tracing** plugin this is what we found out:
- We use RabbitMQ 3.7.6 running with Erlang 20.3.8.1 on a MBP (2,5 GHz Intel Core i7) and `rabbitmq-perf-test-1.1.0`
- **rabbitmq_tracing** was enabled
- `bin/runjava com.rabbitmq.perf.PerfTest -u test` could produce on average 24k msg/sec
- Define a new trace via the management console. `Name = "tracer-1", Pattern = "#"`
- Message throughput dropped to again to 8 msg/sec


### Rabtap

[Rabtap](https://github.com/jandelgado/rabtap) taps messages being sent to exchanges using RabbitMQ exchange-to-exchange bindings without affecting actual message delivery.

TODO

## Sniff AMQP traffic


### Tracer

```
  [AMQP client]<---->[Tracer]<---->[RabbitMQ server]
```
TODO

### TCPDUMP + Wireshark

TODO
