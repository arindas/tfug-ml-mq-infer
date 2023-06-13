# Message Queues: Function and role in ML inference

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

## Usage model

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
>say 120 ml/sqft.
>
>Now they employ a fleet of drones to regularly take images of the plants in the
>farm in both the control and active areas to keep track of the health of the
>plants. The drones take photographs of the plants in the farm at regular
>distances to cover the entire farm. The trajectory of the drones are
>preprogrammed in such way that each drones always take photographs at some 
>specific positions (marked by GPS) and each photograph contains a maximum of 
>20 leaves. The drones take photographs at regular time intervals (e.g daily)
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
- Disease grade classification for 5 diseases (the grades correspond to
severity)

There are no sufficiently large annotated semantic segmentation dataset that
can be used for production purposes. (Let's say, for this scenario.)

### Solution Architecture

The machine learning engineers provide the following models:
- A leaf object detector
- 5 disease grade classifier models. Each disease grade classifier models has 4
  classes.

The software engineers design the solution with a suite of micro-services and
drone controllers which communicate with each other using the message queue. The
various components of the system are described below:

#### Drone Image Collector

The drone moves to specific position clicks an image, and uploads it to an
S3/GCS bucket at a specific path. Each drone position has a unique identifier.
The image path could be of the form:

```
s3://${STORAGE_BUCKET}/${position_id}/${date}/image.png
```

Next the drone controller produces a message on the "drone_image" topic. The
message simply contains the S3 image object path.

(We assume that the drones are equipped with some form of internet
connectivity.)

(In the following section, when I say that a components consumes a message, if
the message contains an image url, assume that the component also downloads the
image from the url to it's local storage.

Also the service might be use a shared persistent storage service to avoid
hitting S3 every time.)

#### Leaf image cropper

The leaf image cropper crops out the leaves from the drone image using the leaf
object detection model.

The leaf object detection is model is served with a model serving solution like
"Tensorflow serving" or "PyTorch serve". The model serving solution provides the
model inference at specific HTTP or gRPC endpoint.

The leaf image cropper has a consumer to consume drone image from the
"drone_image" topic. It then perform inference with the HTTP or gRPC endpoint
served by the model serving solution and obtains all the bounding boxes. It then
uses bounding boxes to crop out the leaves. 

For each cropped out leaf:
- A new cropped leaf image is created. It is uploaded to:
```
s3://${STORAGE_BUCKET}/${position_id}/${date}/${cropper-model-id}/${cropper-model-version}/${leaf_crop_idx}.png
```
- It produces a message on the "cropped_leaf" topic. The message contains the
above S3 image object path.

#### Disease demultiplexer service

This service simply consumes a message from the "cropped_leaf" and produces it
on each of the 5 disease grader topics "disease_0", "disease_1", ...,
"disease_4".

This service might not be necessary in some message queues where they provide
utilities to do this automatically.

#### Individual disease grader services

Each disease has 4 grades with 0 for healthy and 4 for most severe.

A disease grader service uses it's corresponding disease grading model to grade
the severity of the cropped leaf for the corresponding disease.

The "disease_#n" grader model is served with a model serving solution like
"Tensorflow serving" or "PyTorch serve". The model serving solution provides the
model inference at specific HTTP or gRPC endpoint.

A disease grader service consumes a cropped leaf image from it's corresponding
input topic, performs inference with the HTTP or gRPC endpoint exposed by the
model serving solution, and writes a record with the following schema to a
database:

```
{
  position_id: …, // drone position on field
  date: …, // date on which image was taken
  leaf_crop_idx: …, // leaf crop index from leaf crops with obj. det. model
  disease_type: …, // disease grader disease type
  predicted_grade: …, // predicted disease severity grade
  cropper_model_id: …, // model used for cropping
  cropper_model_version: … // version of cropper model used
  disease_model_id: …, // model used for predicting disease
  disease_model_version: …, // version of the disease predictor model used
}
```

#### Analysis Service

This service runs DB analytical queries to produce updated dashboards and
reports for stakeholders in this undertaking. This service runs at regular
intervals daily.

The results of frequently used analytical queries are cached to minimize waiting
times.

#### Audits and Reproducibility

You might have observed that we also store the model id and version with each
record. This is necessary if we need to re-analyze with different versions of
different models.

We can regenerate all the messages by walking over the contents of the message
queue and the system will perform inference as necessary. However, some message
queues (like Apache Kafka) also allow you to replay messages.

For instance, consider we updated the model for "disease_3" and we want to
re-analyze the images. We may simply replay the messages in the topic for
"disease_3" along with new model config for the corresponding disease grader
service. This will produce new disease inference records with the appropriate
model details.

#### Scaling up horizontally

Now as discussed toward the start of the talk, individual services can be
horizontally scaled easily. In our scenario the object detection service could
using a model like MobileNet which has faster inference times, but the classifier
service might be using a UNet for classification which has slower inference
times.

In this case based on the requirements we can scale up the affected service as
necessary. Now since the web component and the actual model serving solution
communicate over the network, they could be scale up independently if necessary.

For instance, we might not choose to scale up the web component of a disease
grader service, but we can increase the number of model replicas in the model
serving solution instead (PyTorch serve for instance.) Also it is not necessary
that the web components each have their unique instance of the model serving
solution. They could all make requests to shared model serving solution
instance.

#### Support for multiple environments

Now in this design itself, there is not strict requirement to specify that the
services are containers, pod in k8s or simple linux processes. As long as they
communicate with each other on the message queue, it doesn't matter where they
run.

This approach in turn allows us to pursue multi-cloud architectures where some
components could be running on AWS, some on GCP some on Azure and so on. At the
same time, this entire system could simply run on just a beefy laptop.
