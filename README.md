# Sending Syslog via Kafka into Graylog

If your setup need buffers on the transport, or your Graylog Server is not reachable from all Network Segments, you can use [Apache Kafka](http://kafka.apache.org) for this. Please be aware Graylog will connect to your [zookeper](https://zookeeper.apache.org) and fetch the topics by regex. Adding SSL or authentification Information is not possible with the current stable Version of Graylog (2.1 at the time of writing).

```
This Guide will not give you a complete copy&paste how-to,
but it will guide you and provide additional information.

Please do not follow the steps if you did not know how to deal
with common issues yourself.   
```


In this scenario a Syslog message will have the following stages:

- send from rsyslog to [logstash](https://www.elastic.co/products/logstash) via TCP/UDP
- send from logstash to Kafka
- consumed by graylog from Kafka
- Syslog extracted from JSON by Graylog

If you run rsyslog above Version 8.7 the message can have the following stages:

- send from rsyslog to kafka
- consumed by graylog from Kafka
- Syslog extracted from JSON by Graylog

We will assume that you have a Kafka running on **kafka.int.example.org (192.168.100.10)** and your Graylog Instance is running on **graylog.int.example.org (192.168.1.10)**. Additional we have the Linux System **syslog.o1.example.org (192.168.50.30)** and **syslog.o2.example.org (192.168.2.30)** that will send Syslog Data. All Systems are running *ubuntu*, so you might need to adjust some configuration path settings.

## prepare Kafka

If you do not have a Kafka Cluster present, you can follow [the quickstart guide](http://kafka.apache.org/documentation.html#quickstart). But please be aware that this is not a production-ready setup!

## send messages with rsyslog
With rsyslog, you can use templates to format messages. Formatting the messages direct at the source will help to have a clean, predictable workflow.

To identify the messages with the Full Qualified Domain Name of the System that has created the message we use the Option ``PreserveFQDN`` - but you will need to have a clean working hostname resolution.

rsyslog will send the message via UDP to the local running logstash.

```
PreserveFQDN on
template(name="ls_json"
         type="list"
         option.json="on") {
           constant(value="{")
             constant(value="\"@timestamp\":\"")     property(name="timereported" dateFormat="rfc3339")
             constant(value="\",\"@version\":\"1")
             constant(value="\",\"message\":\"")     property(name="msg")
             constant(value="\",\"host\":\"")        property(name="hostname")
             constant(value="\",\"severity\":\"")    property(name="syslogseverity-text")
             constant(value="\",\"facility\":\"")    property(name="syslogfacility-text")
             constant(value="\",\"programname\":\"") property(name="programname")
             constant(value="\",\"procid\":\"")      property(name="procid")
           constant(value="\"}\n")
         }

*.* @127.0.0.1:5514;ls_json
```

The configuration above need to be placed inside the ``/etc/rsyslog.d/90-logstash.conf`` on **syslog.o1.example.org** and **syslog.o2.example.org** in our example and rsyslog need to be restarted (``service rsyslog restart``).

## route messages with rsyslog

If you have rsyslog newer than 8.7.0 you can use rsyslog [omkafka](http://www.rsyslog.com/doc/master/configuration/modules/omkafka.html) module to send the messages direct from rsyslog to Kafka.

```
$ModLoad omkafka
action(type="omkafka" topic="logs" broker=["192.168.100.10:9092"] template="ls_json")
```


## route messages with logstash

If your rsyslog does not provide the Kafka Modul, you can use logstash to forward.

Logstash will listen on *localhost* port *udp/5514* for the messages that are coming from rsyslog and forward them to the Kafka cluster.

```
input {
    UDP {
        port => 5514
        host => "127.0.0.1"
        type => syslog
        codec => "json"
        }
}

output {
           kafka {
               bootstrap_server => "192.168.100.10:9092"
               topic_id => "logs"
           }
}
```

Additional configuration can is [in the Kafka module documentation](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-kafka.html) of logstash.


## consume messages with graylog
Now the Data need to be consumed by graylog. Create an [input](http://docs.graylog.org/en/2.0/pages/getting_started/config_input.html) with the Input *Syslog Kafka*. Add the Information that you configured in the former steps (exchange, username, password, hostname). Set the Option *Allow overwrite date*.

Start the Input to consume the first messages and create [a JSON extractor](http://docs.graylog.org/en/2.0/pages/extractors.html#using-the-json-extractor). Additional create a second extractor on the field `host` and the type `copy input` and store it in the field `source`. You might want a third `copy input` to store `@timestamp` in `timestamp`.

## what's next?
Use the *rsyslog* Systems as Syslog Proxies for every possible source in the same network, add more systems to your setup.


# Credits
- untergeek for [rsyslog / json template](https://gist.github.com/untergeek/0373ee85a41d03ae1b78) and the [blogpost](http://untergeek.com/2012/10/11/using-rsyslog-to-send-pre-formatted-json-to-logstash/)
- IETF for [documentation ips](https://tools.ietf.org/html/rfc5737)
