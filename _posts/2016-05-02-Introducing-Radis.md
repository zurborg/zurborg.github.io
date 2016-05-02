---
layout: post
title: Introducing Radis
---

A long time ago, I went out to find a centralized logging solution.  This
artical is about where I finally arrived.

First, a note about the infrastructure I take care of: the underlying
network layout is a load balancing server, connected to the internet, with
many backend servers (most of them running Apache HTTPd), some database
nodes in a cluster and some special purpose hosts.  And one host is supposed
for centralized logging.

## Market analysis

The first step was a very classical market analysis.  Besides syslog, I've
first heard of [Graylog](http://graylog.com/), but I found some more
solutions:

* [Logstash](https://www.elastic.co/products/logstash)
* [Apache Kafka](https://kafka.apache.org/)
* [Scribe](https://github.com/facebookarchive/scribe)

Graylog was my favourite choice, and still is.  In my earlier days I worked
at [bytemine](https://bytemine.net) together with [Bernd
Ahlers](https://github.com/bernd).  That was the first time I've heard of
centralized logging at all.  Today Bernd is working for Graylog and Icinga.

## Trial and error

So I got Graylog a try.  The first shock was: it is a huge Java application. 
I'm not telling about that I don't like Java, but Java applications needs a
huge amount of memory.  That should be considered.  The next surprise was
the database backend of Graylog: Elasticssearch.  Things get worse, it is a
Java application too.  And it needs much more memory then Graylog.

But I finally got them installed and working.  Graylog has a really nice web
frontend, but also the frontend needs not-so-little amount of memory.  Not
because of Java but of JavaScript.  Hint: the frontends works very fast in
Google Chrome.  Things are not so bad as they seem.

## How to put data into Graylog, the easy way

Graylog has an integrated syslog server, listening on both TCP and UDP. 
I've managed some servers putting messages via rsyslog into Graylog.  It
works out of the box and I'm very happy not touching any config files for
that.

But then I've read more articles about logging.  Syslog is an old-fashioned
protocol, and I was advised to avoid Syslog whenever possible.  Graylog itself
uses GELF messages, which is simply a JSON string containing some mandatory
keywords.

## How to put data into Graylog, the bloated way

Okay, generating GELF messages on the one hand.  That could be managed by
each application on its own way.  Not really standardized, but GELF is open
enough to get almost everything you want in a logging message.  I've looked
at many examples how to do so and many examples still using Syslog for
transport.

The earlier version of graylog provides a helper tool, called
_graylog-radio_ which managed a reliable queue.  Today it is part of the big
Graylog server application.  To accomplish a really good setup with reliable
queues on each server you need to install this huge Java software on _every_
server.

I'm not going to do this. So I had to start a market analysis again.

## Reliable logging

Two technologies gaining reliable queues are AMQP and Kafka, for example.

Apache Kafka needs Java so it's out of the race.
[RabbitMQ](https://www.rabbitmq.com/) is written in Erlang (that's okay for
me) but hard to configure.  I've looked at some other solutions, too, but
they are either Java bloatware or really hard to configure.  Finally, I
didn't managed it.  What I need is a really simple software, caching the
GELF messages in a reliable queue, in order to push them into Graylog
directly or wait until the server becomes available.

Time passes by and I worked on another project that heavily uses a Redis DB. 
I read some notes about creating reliable queues in a Redis database using a
special command named
[RPOPLPUSH](http://redis.io/commands/rpoplpush#pattern-reliable-queue).

Finally I've got an idea: what about putting GELF messages into Redis?

## Putting things together

Thats not hard at all.  Using simple technologies is very important in such
concepts.  We want reliable logging, over a network, never missing any
message, even if the Graylog cluster or network is down for a while.  These
things _are_ simple, compared to AMQP and Kafka:

* GELF, a JSON based log format, with 3 required fields and 2 more for a
  verbose message
* Redis, a in-memory key-value database, with support for
  high-availbility cluster setups and persitent storage
* A helper script, generating GELF messages and putting them with a single
  command into Redis
* A helper daemon, listening to the queue in Redis DB and piping them to
  Graylog via TCP

So I started with the latter task and wrote a Perl script.  It does just
what I said above: listening to _one_ Redis DB and piping messages to _one_
graylog server.  In future, I have to add some more options to support
cluster setups, for example.  But from now on my script works and fits my
needs.

The other task sounds simple but is very tricky.  Have you ever told Apache
HTTPd to generate a JSON string as logging output?  So I need a proxy
script.  Apache allows me to pipe log messages to a programm.  I wrote a
second Perl script, reading from standard input and create GELF messages. 
That time I played with it a bit too much and created a new log format,
fitting my needs.  But in the end, I don't think it's imporant that these
short-living messages from Apache to my helper script must be
human-readable.  They aren't.  By the way, a single GELF message is almost
human-readable, but many messages almost not.

## Client libraries

After that horrible trip with Apache, there are also good stories.  When I
wanted to adapt my idea to frameworks like [Dancer](http://perldancer.org/),
[Cake](http://cakephp.org/), [Yii](http://www.yiiframework.com/) or even
[Joomla](http://www.joomla.org/), I've got very good documentated APIs
allowing me to easily implement a suitable wrapper.

The example code in Perl:

```perl
use Redis;
use JSON qw(encode_json);
use Sys::Hostname qw(hostname);

# Connect to Redis DB
my $redis = Redis->new(server => "localhost:6379");

# Build log message
my $message = {
  'host' => hostname(),
  'timestamp' => time(),
  'short_message' => 'Hello!'
};

# Encode to JSON
my $gelf = encode_json($message);

# Push to queue
$redis->lpush('graylog-queue', $gelf);
```

PHP is quite similiar, needs one depedency less (because JSON is already
part of PHP):

```php
$redis = new Redis();

# Connect to Redis DB
$redis->connect('localhost', 6379);

# Build log message
$message = [
  'host' => gethostname(),
  'timestamp' => time(),
  'short_message' => 'Hello!'
];

# Encode to JSON
$gelf = json_encode($message);

# Push to queue
$redis->lPush('graylog-queue', $gelf);
```

And even shell scripting:

```bash
echo 'lpush graylog-queue {"host":"'$(hostname)'","timestamp":'$(date +%s)',"short_message":"Hello!"}' | redis-cli
```

Okay, that example has a few disadvantages, but demonstrate how easy the
implementation might be.

## Current status

I named the whole concept *Radis*.  The word _Radis_ is derived from _Redis_
and _Radio_.  Radis is not supposed to be a piece of software, a protocol or
something.  It's just the concept of passing GELF messages into a Redis DB
using the _LPUSH_ command.

A list of software I wrote (and published) for Radis:

* Generic Perl API: `Log::Radis`
  [github](https://github.com/zurborg/liblog-radis-perl)
  [metacpan](https://metacpan.org/pod/Log::Radis)
* Engine for Dancer2: `Dancer2::Logger::Radis`
  [github](https://github.com/zurborg/libdancer2-logger-radis-perl)
  [metacpan](https://metacpan.org/pod/Dancer2::Logger::Radis)
* Engine for CakePHP: `Cake\Log\Engine\Radis`
  [github](https://github.com/zurborg/libcake-log-engine-radis-php)
  [packagist](https://packagist.org/packages/zurborg/cake-log-engine-radis)

## TODO

* Publishing the rest of software I wrote, including:
  * `radis-daemon` which pipes messages form Redis DB to Graylog
  * `apache2radis` a log wrapper for Apache HTTPd

And last but not least I need feedback.  Do you like my idea?  Write me. 
You wanna get involved?  Fork me at github.  Do you think that is all
rubbish and I'm stupid because I wasn't able to install a piece of software? 
Yeah, maybe you're right.  Let's talk about that.
