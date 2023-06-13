# Message Queues and their role in Machine Learning Inference

## Intro to Message Queues

Let's first talk a bit about message queues.

We see the word "queue" in it, why?

![queue-diagram](/some/url)

A queue is a first in first out data-structure for storing a set of elements.

However this definition doesn't tell us anything about:
- How and where the elements are stored (in-memory, hard disk etc.)
- Who can access this queue from where

In the 21st century, we have the following additional requirements:
- It has to be accessible over the internet over well known protocols (HTTP,
  gRPC etc.)
- It has to be persistent if necessary
- It has to be replicated to be highly available.
- All replicas need to be consistent with each other
- It has to support multiple channels for different types of elements
- It has to be reasonably fast
- We can't be bothered to worry about these requirements. We are more worried
  about business requirements

Message queues in the 21st century are queues of "message" elements, and they
fulfil the above requirements.

## Message Queues' usage

In the conventional client server model, the client makes synchronous requests
to the server, which responds back with a synchronous response.

![client-server-model](/some/url)

What happens when we have too many clients that the server can't handle at once?

![client-server-model-overwhelmed](/some/url)

Message queues enable asynchronous handling of requests.

However, this is not the only thing that message queues are good at. Message
queues also enable large scale request handling with _multiple stages_, and
_horizontal scaling_ at each stage. But how?

![multi-stage-process-diagram](/some/url0)

Let's take the example of two popular message queues / messaging services:
- Apache Kafka
- Google Cloud Pub/Sub

### Apache Kafka

In Apache Kafka, messages are organized by "topic"(s). Each "topic" is further
broken down partitions. Messages in a partition are ordered and isolated from
messages in another partition.

![topic-partition-hierarchy](/some/url)

There are two kinds of clients for Apache Kafka: Producers and Consumers

- Producers produce messages into a topic in Kafka. They can choose to write to
  a particular partition or let Kafka load balance messages to a partition.
- Consumers consume message from a particular partition under a topic.

In our multistage computation example, each stage has input request and output
responses. Let's assign a particular topic to a particular stage. To achieve
horizontal scaling we do the following:
- We make multiple partitions under the topic for this stage.
- We write input requests to the topic. The partitioner load balances the
  messages to the partitions under this topic.
- We run multiple instances of the request hander for this stage. Each stage
  contains a consumer which reads from a particular partition.
- In order to horizontally scale up further, we increase the number of
  partitions and request handler instances.


### Google Cloud Pub/Sub

Google Cloud Pub/Sub makes this a bit more simpler. There are "topic"(s) where
messages are written to. Each "topic" can have a set of "subscription"(s). There
can be multiple publishers to a topic, and multiple subscribers that read from
multiple subscriptions from the topic.

Message published to the topic are load balanced across all the subscriptions.


>Some also call this event driven architecture, with the individual messages
>being "event"(s)

## Case Study: Plant medicine effectiveness on Crops

Remember that this talk was actually about Machine Learning inference with
message queues and not just about message queues. To illustrate how message
queues might be used in a machine learning inference system in the real world,
consider the following scenario:

>An agricultural pharmacy company has come up with a new drug to cure various
>plant diseases. Now, they wish to measure the effectiveness of their medicine.
>They want to quantitatively analyze if the medicine speeds up the healing
>process in plants.
>
>In order to to do this, the company partners with a farm owner to test out the
>medicine on their crops. They isolate the farm into two areas, the control area
>where no medicine is used and the active area where the newly developed medicine
>is tested out. They decide on a uniform medicine distribution quantity, let's
>say 270 ml/sqft.
>
>Now they employ a fleet of drones to regularly take images of the plants in the
>farm in both the control and active areas to keep track of the health of the
>plants. The drones take photographs of the plants in the farm at regular
>distances to cover the entire farm. The trajectory of the drones are
>preprogrammed in such way that each drones always take photographs at the same
>position (marked by GPS) and each photograph contains a maximum of 20 leaves.
>The drones take photographs at regular time intervals.
>
>(The plants are in a green house. They don't move around a lot due to absence
>of wind)
>
>The Machine Learning engineers are to analyze the photographs taken by the
>drones and measure the effect of the medicine on the leaves.

Now take a look at this photograph:

![disease-photo](https://media.istockphoto.com/id/483451251/photo/fungal-attack.jpg?s=612x612&w=0&k=20&c=PM0Lld99Io4DU6sRqemkytZUkuSF5effOJ8fhIAXwVo=)

Intuitively it makes sense that if the disease progresses, the disease affected
area will spread. If it heals, the disease area will decrease.

The excellent machine learning engineers in this room would agree that
approaching this problem with semantic segmentation would be powerful. From a
single image with multiple leaves, multiple disease types can be detected along
with the area covered by each type.

However that is an ideal case. The machine learning engineers only have the
following datasets:
- Leaf object detection dataset
