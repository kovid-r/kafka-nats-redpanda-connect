# Using Redpanda Connect to connect NATS and Kafka

Redpanda Connect is a stateless config-based stream processing tool with filtering, transformation, and enrichment capabilities, and it allows you to work with multiple sources and sinks. In this article, we'll use it to get data from Kafka and pass it to NATS JetStream. The following architecture diagram depicts the flow of data from Kafka to NATS JetStream via Redpanda Connect:

![Connecting Apache Kafka and NATS using Redpanda Connect - Image by author](https://i.imgur.com/dIesh27.png)

To reiterate, here are the three critical components in the above diagram:

* [Apache Kafka](http://kafka.apache.org) - producer of events
* [NATS](http://nats.io) - consumer/distributor of events
* [Redpanda Connect](https://redpanda.com/blog/redpanda-connect) - bridge/connector between Apache Kafka and NATS, with additional data processing responsibilities.

Now that the architecture is clear, let's get into the installation and configuration of these tools.

## Connecting NATS and Kafka

As shown in the diagram in the previous section, Redpanda Connect will enable the connection between NATS and Kafka. However, before doing that, you'll need to install Apache Kafka and a NATS server with JetStream enabled. Let's get into it.

### Install NATS server and set up NATS JetStream

To use NATS, you'll first need to install a NATS server with JetStream enabled. You can use the following `curl` command to download and install the NATS server on your machine:

```shell
curl -sf https://binaries.nats.dev/nats-io/natscli/nats | sh
```

You can now start the NATS server with JetStream enabled and the storage directory (where stream data gets stored) set to your current folder:


```shell
./nats-server -js --store_dir .
```

Run the following health check command to ensure everything is OK with your NATS server:
        
```shell
nats server check connection -s 0.0.0.0
```

While in this article, you'll use the self-hosted version of the NATS server, when you want to deploy a highly available and highly scalable solution, you can use [Synadia Cloud](https://www.synadia.com/cloud). This is a fully managed global service that takes care of infrastructure, account and user management, and JetStream management, among various other things. You can read more about how Synadia Cloud is [helping the automotive industry](https://www.synadia.com/solutions/automotive) by significantly improving connectivity between vehicles, operators, fleets, road networks, and so on.

### Install and set up Kafka

Next, let's set up Kafka. You can download the latest Kafka binaries from the [official website](https://www.apache.org/dyn/closer.cgi?path=/kafka/3.7.1/kafka_2.13-3.7.1.tgz) and run the following commands to install them on your system:

```shell
tar -xzf kafka_2.13-3.7.1.tgz
cd kafka_2.13-3.7.1
```

If you're on a Mac, you can use `brew` to do the installation.

```shell
brew install kafka
```

Start the Kafka server by pointing it to the default `server.properties` file:

```shell
/opt/homebrew/Cellar/kafka/3.7.1/libexec/bin/kafka-server-start.sh server.properties
```

Once the server has started, create a topic, which, in this case, is called `kafka-nats` using the following command:

```shell
kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic kafka-nats
```

This becomes the producer of events, a.k.a. the source for [Redpanda Connect](https://redpanda.com/blog/redpanda-connect) and NATS, which we'll come to later.

Open another tab in your terminal window and create a consumer of the messages published on the topic you just created:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --topic kafka-nats --from-beginning
```

Please note that creating the consumer is optional. You should only do this to test whether the publishing and consumption are working properly within Kafka. Once the validation is done, NATS should be ready to consume the messages on the topic via Redpanda Connect.
       
### Install Redpanda Connect

To use Benthos.dev, now known as Redpanda Connect, let's download it using the following `curl` command:
       
```
curl -LO https://github.com/redpanda-data/redpanda/releases/latest/download/rpk-darwin-arm64.zip
```

Please note that the command mentioned above will differ for different operating systems. 

To install Redpanda Connect, run the following commands that create a new directory, export the path of that directory to the `PATH` environment variable in the terminal, and install the `rpk` command-line client on your machine:

```
mkdir -p ~/.local/bin
export PATH=$PATH:~/.local/bin
unzip rpk-darwin-arm64.zip -d ~/.local/bin/
```

To make sure that the installation has succeeded, please run the following command to check the version of the client:

```
rpk --version
```

Now that NATS, Kafka, and Redpanda Connect have been installed, the only thing that remains for data to flow between Kafka and NATS is the Redpanda Connect configuration, as described in the architecture diagram at the beginning of the article. In this configuration, Kafka is configured as a source, i.e., input, and NATS is configured as a sink, i.e., output.

### Configure Redpanda Connect

Let's start by configuring the input. To configure Kafka as a source, you need to provide the following three things:

* `address` - where Kafka's bootstrap server is hosted. In this case, it is `localhost:9092`.
* `topics` - the topic(s) you want Redpanda Connect to consume and process. In this case, we only have one topic, i.e., `kafka-nats`.
* `consumer_group` - Consumer groups are used to horizontally scale consumption from topics. As we're not testing for scale in this example, let's not concentrate on it too much and name it `kafka-nats-group`.

All of these three fields are mandatory. The rest, which you can find in the [official documentation](https://docs.redpanda.com/redpanda-connect/components/inputs/about/), are optional.

Now that you have the input as Kafka sorted out, set up the pipeline that extracts the following from the source JSON messages:

* `company` - name of the company
* `name` - the person's name who placed the order
* `email` - hashed email of the person who placed the order (masked using the SHA256 encryption algorithm)
* `orderdate` - the date of the order

Now, you'll need to configure the NATS JetStream output, a.k.a, sink using the following two fields:

* `urls` - the full address where the NATS JetStream server listens for events. This is the address in the terminal message when you start the NATS JetStream server. In this case, the address is `nats://127.0.0.1:4222`
* `subject` - as it is a mandatory field, you can use an arbitrary subject name but this article doesn't use subject hierarchies in NATS JetStream.

Using all the information mentioned above about the input, the pipeline, and the output, you'll end up with a Redpanda Connect YAML-based configuration file that looks like the following:

```yaml
input:
  kafka:
    addresses:
      - localhost:9092
    topics:
      - kafka-nats 
    consumer_group: kafka-nats-cg

pipeline:
  processors:
    - sleep:
        duration: 50ms
    - mapping: |
        root.company = this.data.company.name
        root.name = this.data.company.contactperson.name
        root.email = this.data.company.contactperson.email.hash("sha256")
        root.orderdate = this.data.dateoforder

output:
  nats_jetstream:
    urls: [ "nats://127.0.0.1:4222" ]
    subject: "kafka-nats-subject"
```

Using this file and the following command, you can start the connection between Apache Kafka and NATS:

```shell
rpk connect -c connect.yaml
```

You should now see the following messages on the terminal screen:

```shell
INFO Running main config from specified file       @service=benthos benthos_version=24.1.9 path=connect.yaml
INFO Launching a Redpanda Connect instance, use CTRL+C to close  @service=benthos
INFO Output type nats_jetstream is now active      @service=benthos label="" path=root.output
INFO Input type kafka is now active                @service=benthos label="" path=root.input
```

This means that the connection has been successful. To test the setup, publish individual lines in the following array as JSON messages in Kafka.

```json
[
    {"data":{"company":{"name":"JD Inc.","contactperson":{"name":"John Doe","email":"john.doe@jdmail.com"}},"price":"1050.00","dateoforder":"12/07/2024"}},
    {"data":{"company":{"name":"Smith Enterprises","contactperson":{"name":"Jane Smith","email":"jane.smith@semail.com"}},"price":"1200.00","dateoforder":"15/07/2024"}},
    {"data":{"company":{"name":"Tech Solutions","contactperson":{"name":"Mark Brown","email":"mark.brown@tsmail.com"}},"price":"980.00","dateoforder":"18/07/2024"}},
    {"data":{"company":{"name":"Innovate LLC","contactperson":{"name":"Lucy Green","email":"lucy.green@illcmail.com"}},"price":"1150.00","dateoforder":"20/07/2024"}},
    {"data":{"company":{"name":"Global Corp","contactperson":{"name":"Michael White","email":"michael.white@gcmail.com"}},"price":"1300.00","dateoforder":"22/07/2024"}},
    {"data":{"company":{"name":"Future Tech","contactperson":{"name":"Laura Blue","email":"laura.blue@ftmail.com"}},"price":"1075.00","dateoforder":"25/07/2024"}},
    {"data":{"company":{"name":"Bright Ideas","contactperson":{"name":"David Black","email":"david.black@bimail.com"}},"price":"1250.00","dateoforder":"28/07/2024"}},
    {"data":{"company":{"name":"NextGen Inc.","contactperson":{"name":"Olivia Gray","email":"olivia.gray@ngmail.com"}},"price":"1100.00","dateoforder":"30/07/2024"}},
    {"data":{"company":{"name":"Prime Solutions","contactperson":{"name":"Paul Brown","email":"paul.brown@psmail.com"}},"price":"995.00","dateoforder":"02/08/2024"}},
    {"data":{"company":{"name":"Smart Innovations","contactperson":{"name":"Emily White","email":"emily.white@simail.com"}},"price":"1400.00","dateoforder":"05/08/2024"}},
    {"data":{"company":{"name":"Elite Enterprises","contactperson":{"name":"James Black","email":"james.black@eemail.com"}},"price":"1180.00","dateoforder":"08/08/2024"}}
]
```

Note how the source JSON message has a nested structure and the email is unmasked. You can check the output by going to a new terminal window and typing in the following command:

```shell
nats stream view
```

You can also see the summary of all these messages from NATS using the following command:

```shell
nats stream report
```

The command above has the following output:

```shell
Obtaining Stream stats

╭───────────────────────────────────────────────────────────────────────────────────────────────╮
│                                         Stream Report                                         │
├────────────┬─────────┬───────────┬───────────┬──────────┬─────────┬──────┬─────────┬──────────┤
│ Stream     │ Storage │ Placement │ Consumers │ Messages │ Bytes   │ Lost │ Deleted │ Replicas │
├────────────┼─────────┼───────────┼───────────┼──────────┼─────────┼──────┼─────────┼──────────┤
│ kafka-nats │ Memory  │           │         0 │ 11       │ 1.7 KiB │ 0    │       0 │          │
╰────────────┴─────────┴───────────┴───────────┴──────────┴─────────┴──────┴─────────┴──────────╯
```

As you can see in the report above, 11 messages were passed on from Kafka to Redpanda Connect for processing and handover to the NATS server. If the Redpanda Connect configuration worked correctly, the collated output should look something like the following:

```json
[
  {"company":"JD Inc.","email":"/y2xKnsBxZ9svYXLJYreF+KvZcGz30sSCii0dOt5RHs=","name":"John Doe","orderdate":"12/07/2024"},
  {"company":"Smith Enterprises","email":"J/XjAZzFMk1wE5b4m/G/T5uOAeRd6ep/TNUw51RA8Rs=","name":"Jane Smith","orderdate":"15/07/2024"},
  {"company":"Tech Solutions","email":"HFxJTvcjZ+Oq4VEAn2In9doLTGTekKIlA4F9WmSceAY=","name":"Mark Brown","orderdate":"18/07/2024"},
  {"company":"Innovate LLC","email":"g2Kov5zhGrqBF6D4DnX/vG6qkby5w1cpvGO9W0ZVHow=","name":"Lucy Green","orderdate":"20/07/2024"},
  {"company":"Global Corp","email":"kTintSH97blGqknuTP4mJahDEdYR0upEqBs7/sS+Coc=","name":"Michael White","orderdate":"22/07/2024"},
  {"company":"Future Tech","email":"cF2sFbDnLWrWwg7LHedliIU4MFBSTBtT2MItO3/FXxQ=","name":"Laura Blue","orderdate":"25/07/2024"},
  {"company":"Bright Ideas","email":"nR80FyCZTH/hvNb+7v5Nj2l9ShrNuWnjMdh+omhAHuU=","name":"David Black","orderdate":"28/07/2024"}, 
  {"company":"NextGen Inc.","email":"lG+gaAuCan4yptV7AFNe5k+Pfne1+qunin4BGN6k38M=","name":"Olivia Gray","orderdate":"30/07/2024"},
  {"company":"Prime Solutions","email":"jMqGCWexvhOMRGfYQJE0VrQE6uzVNjT1LhGA+EYNziM=","name":"Paul Brown","orderdate":"02/08/2024"},
  {"company":"Smart Innovations","email":"imyi4RNgnsQLs1T2HyhWM58cGhEL+Sg9eP6Sqaf10h8=","name":"Emily White","orderdate":"05/08/2024"}
]
```

Messages from Apache Kafka can now flow to NATS via the pipeline enabled by Redpanda Connect.
