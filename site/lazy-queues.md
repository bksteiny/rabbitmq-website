# Lazy Queues

## <a id="overview" class="anchor" /> [Overview](#overview)

Since **RabbitMQ 3.6.0**, the broker has the concept of **Lazy Queues** -
queues that move their contents to disk as early as practically possible,
and only load them in RAM when requested by consumers,
therefore the lazy denomination.

One of the main goals of lazy queues is to be able to support very
long queues (many millions of messages).  Queues can become very long
for various reasons:

<ul class="plain">
  <li>consumers are offline / have crashed / are down for maintenance</li>
  <li>there is a sudden message ingress spike, producers are outpacing consumers</li>
  <li>consumers are slower than normal</li>
</ul>

By default, queues keep an in-memory cache of messages that is filled up as messages are published into RabbitMQ.
The idea of this cache is to be able to deliver messages to consumers as fast as possible.
Note that persistent messages can be written to disk as they enter the broker **and** kept in RAM at the same time.

Whenever the broker [considers it needs to free up memory](memory.html), messages from this cache will be [paged out to disk](persistence-conf.html).
Paging a batch of messages to disk takes time and blocks the queue process,
making it unable to receive new messages while it's paging.
Even though recent versions of RabbitMQ improved the paging algorithm,
the situation is still not ideal for use cases where you have
many millions on messages in the queue that might need to be paged out.

Lazy queues attempt to move messages to disk as early as practically possible.
This means significantly fewer messages are kept in RAM in the majority of cases under normal operation.
This comes at a cost of increased disk I/O.

## <a id="configuration" class="anchor" /> [Making a Queue Lazy](#configuration)

Queues can be made to run in `default` mode or `lazy` mode by:

<ul class="plain">
  <li>setting the mode via <code>queue.declare</code> arguments</li>
  <li>applying a queue <a href="parameters.html#policies">policy</a></li>
</ul>

When both a [policy](parameters.html) and queue arguments specify a queue mode, the queue argument has priority over the policy value.

If a queue mode is set via an optional argument at the time of declaration,
it can only be changed by deleting the queue, and re-declaring it later with a different argument.

### Using Arguments at the Time of Declaration

The queue mode can be set by supplying the `x-queue-mode` queue declaration argument with a string specifying the desired mode.
Valid modes are:

<ul class="plain">
  <li><code>"default"</code></li>
  <li><code>"lazy"</code></li>
</ul>

If no mode is specified during declare, then `"default"` is assumed.
The `default` mode is the behaviour already present in pre 3.6.0 versions of the broker,
so there are no breaking changes in this regard.

This example in Java declares a queue with the queue mode set to `"lazy"`:

<pre class="sourcecode java">
  Map&lt;String, Object> args = new HashMap&lt;String, Object>();
  args.put("x-queue-mode", "lazy");
  channel.queueDeclare("myqueue", false, false, false, args);
</pre>

### Using a policy

To specify a queue mode using a policy, add the key `queue-mode` to a policy definition, e.g.:

<table>
  <tr>
    <th>rabbitmqctl</th>
    <td>
      <pre>rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"lazy"}' --apply-to queues</pre>
    </td>
  </tr>
  <tr>
    <th>rabbitmqctl (Windows)</th>
    <td>
      <pre>rabbitmqctl set_policy Lazy "^lazy-queue$" "{""queue-mode"":""lazy""}" --apply-to queues</pre>
    </td>
  </tr>
</table>

This ensures the queue called `lazy-queue` will work in the `lazy` mode.

Policies can also be defined via [management UI](management.html).

### Changing Queue Mode at Runtime

If queue mode was configured via a policy,
it is possible to change it at runtime without the need of deleting and re-declaring the queue.
To make a queue named "lazy-queue" use the default (non-lazy) mode, update its matching policy
to specify a different `queue-mode`:

<table>
  <tr>
    <th>rabbitmqctl</th>
    <td>
      <pre>rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"default"}' --apply-to queues</pre>
    </td>
  </tr>
  <tr>
    <th>rabbitmqctl (Windows)</th>
    <td>
      <pre>rabbitmqctl set_policy Lazy "^lazy-queue$" "{""queue-mode"":""default""}" --apply-to queues</pre>
    </td>
  </tr>
</table>

## <a id="performance" class="anchor" /> [Performance Considerations for Lazy Queues](#performance)

### Disk Utilization

A lazy queue will move its messages to disk as soon as practically possible, even if the message was published
as transient by the publisher. This generally will result in higher disk I/O utilisation.

A regular queue will [keep messages in memory for longer](memory.html).
This will result in delayed disk I/O which is less even (has more spikes)
since more data will need to be written to disk at once.

### RAM Utilization

While it's impossible to provide accurate numbers every use
case, this is a simplistic test that showcases the difference in RAM
utilization between a regular &amp; a lazy queue:

| Number of messages | Message body size | Message type | Producers | Consumers |
| -                  | -                 | -            | -         | -         |
| 1,000,000          | 1,000 bytes       | persistent   | 1         | 0         |

The RAM utilization for default &amp; lazy queues **after** ingesting the above messages:

| Queue mode | Queue process memory | Messages in memory | Memory used by messages | Node memory |
| -          | -                    | -                  | -                       | -           |
| `default`  | 257 MB               | 386,307            | 368 MB                  | 734 MB      |
| `lazy`     | 159 KB               | 0                  | 0                       | 117 MB      |

Both queues persisted 1,000,000 messages and used 1.2 GB of disk space.

#### Test Details

Below is a transcript of the test performed with a queue in the regular (default) mode:

<pre class="sourcecode bash">
# Start a temporary RabbitMQ node:
#
#       export RABBITMQ_NODENAME=default-queue-test
#       export RABBITMQ_MNESIA_BASE=/tmp
#       export RABBITMQ_LOG_BASE=/tmp
#       rabbitmq-server &amp;
#
# (the last command will fail if there is another RabbitMQ node already running)

# In a https://github.com/rabbitmq/rabbitmq-perf-test clone, run:
make run ARGS="-y 0 -s 1000 -f persistent -C 1000000 -u default -ad false"
# Run gmake on OS X

# Queue stats:
rabbitmqctl list_queues name arguments memory messages_ram message_bytes_ram messages_persistent message_bytes_persistent
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
default	[]	417421592	386307	386307000	1000000	1000000000

# Node memory stats
rabbitmqctl status | grep rss,
      {total,[{erlang,1043205272},{rss,770306048},{allocated,1103822848}]}]},

# Stop our temporary RabbitMQ node &amp; clean all persistent files
#
#       rabbitmqctl shutdown
#       rm -fr /tmp/{log,$RABBITMQ_NODENAME*}
</pre>

With a lazy queue the transcript is very similar:

<pre class="sourcecode bash">
# Use a different RABBITMQ_NODENAME
# All other variables remain the same as the previous example
#
#       export RABBITMQ_NODENAME=lazy-queue-test

# In a https://github.com/rabbitmq/rabbitmq-perf-test clone, run:
make run ARGS="-y 0 -s 1000 -f persistent -C 1000000 -u lazy -qa x-queue-mode=lazy -ad false"
# Run gmake on OS X
</pre>

**Note that this was a very simplistic test.** Please make sure to do benchmarks for your specific
workload and use this test as a starting point.


### Switching Queue Mode at Runtime

When converting a `default` queue into a `lazy` one,
the operation will suffer the same performance impact as when a queue needs to page messages to disk.

During a conversion from the regular mode to the lazy one,
the queue will first page all messages kept in RAM to disk. It won't accept any more
messages from publishing channels while that operation is in progress.
After the initial pageout is done, the queue will start accepting publishes, acks, and other commands.

When a queue goes from the lazy mode to the default one,
it will perform the same process as when a queue is recovered after a server restart:
a batch of 16384 messages will be loaded into memory.


## <a id="caveats-limitations" class="anchor" /> [Caveats and Limitations](#caveats-limitations)

Lazy queues are appropriate when keeping node memory usage low is a priority
and higher disk I/O and disk utilisation are acceptable. Lazy queues have other aspects
that should be considered.

### Node Startup

When a node is running and under normal operation, lazy queues will keep all messages on disk,
the only exception being in-flight messages.

When a RabbitMQ node starts, all queues, including the lazy ones, will load up to **16,384** messages into RAM.
If [queue index embedding](persistence-conf.html) is enabled (the `queue_index_embed_msgs_below` configuration parameter is greater than 0),
the payloads of those messages will be loaded into RAM as well.

For example, a lazy queue with **20,000** messages of **4,000** bytes each, will load **16,384** messages into memory.
These messages will use **63MB** of system memory.
The queue process will use another **8.4MB** of system memory, bringing the total to just over **70MB**.

This is an important consideration for capacity planning if the
RabbitMQ node is memory constrained, or if there are many lazy queues
hosted on the node.

**It is important to remember that an under-provisioned RabbitMQ node in terms of memory or disk space will fail to start.**

Setting `queue_index_embed_msgs_below` to `0` will disable payload embedding in the queue index.
As a result, lazy queues will not load message payloads into memory on node startup.
See the [Persistence Configuration guide](persistence-conf.html) for details.



#### Lazy Queues with Mixed Message Sizes

If all messages in the first **10,000** messages are below the
`queue_index_embed_msgs_below` value, and the rest are above this
value, only the first **10,000** will be loaded into memory on node
startup.

#### Lazy Queues with Interleaved Message

Given the following interleaved message sizes:

| Position in queue | Message size in bytes |
| -                 | -                     |
| 1                 | 5,000                 |
| 2                 | 100                   |
| 3                 | 5,000                 |
| 4                 | 200                   |
| ...               | ...                   |
| 79                | 4,000                 |
| 80                | 5,000                 |

Only the first **20** messages below the `queue_index_embed_msgs_below` value will be loaded into memory on node startup.
In this scenario, messages will use **21KB** of system memory, and queue process will use another **32KB** of system memory.
The total system memory required for the queue process to finish starting is **53KB**.