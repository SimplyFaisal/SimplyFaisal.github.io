Title: flexible work queues with rabbitmq and protocol buffers
Date: 2017-07-01 10:20
Category: Review

## Background

I was working on a simple Node.js application a few weeks ago that involved
performing some long(er) running tasks in the background. Specifically, I wanted to send emails and upload documents to S3. I had dabbled with RabbitMQ in the
past and I figured that this was a good use case.

In its simplest usage RabbitMQ sends blobs of data between two recipients, the producer and the consumer. This is a gross simplification, definitely check out the [documentation](https://www.rabbitmq.com/tutorials/tutorial-one-python.html) to see the full extent of its capabilities. In my case I wanted the producer to encode a unit of work that will be read and performed by a consumer. With respect to sending emails in the background I wanted to specify the template to use and the objects needed to render that particular template. This is relatively simple to do in Nodejs. You just have to create an object that holds whatever data that you need, coax it to a string, forward it to an exchange(or drop it directly into a queue), and deserialize it when it arrives at a consumer.

## Issues

After doing this I began ruminating on the toy problem of making my current set
up a bit more versatile. Some of the problems that plagued my current
implementation were:

 1. Serializing and deserializing JSON is simple in Javascript. This cannot be
said for other languages(I'm looking at you Java).

 2. JSON does not provide strong guarantees as to the structure of its contents.
 Consumers would have to perform tedious checks to ensure any fields needed to
 perform a task are indeed present.

## Potential Solution

Protocol Buffers were the perfect solution to the above problems. Other data serialization formats like Apache Thrift or Apache Avro would also suffice. The basic idea is to first define the unit of work
in a language agnostic manner using either Protocol Buffers
or similar technology. Then in your producers you can populate the fields, serialized them, pass them to RabbitMQ and finally deserialize them at the consumers. The consumer can comfortably go about its business safe in the guarantees that Protocol Buffers provide.

## Scenario

Freshman year I wrote a course registration bot to sign me up for classes. I had always nurtured dreams of turning it into a side gig by packaging it as a webapp and selling it to my less technical classmates for a pretty penny. For our particular scenario say we've successfully registered a student for a course and we would like to send an email to notify them. Let's see how we can implement a
simple work queue using RabbitMQ and Protocol Buffers.

## Implementation

First we need to define our "domain" using proto definitions. Below is a sample
.proto file for our scenario.

```proto
syntax = "proto3";

message Student {
        string email = 2;
}

message Course {
        string name = 1;
}

message EmailStudentTask {
        Student student = 1;
        Course course = 2;
}
```

Next we implement our producer and consumer in our language(s) of choice. In our case python and Java respectively:

email_producer.py

```python
import pika
import random
import time
from tutorial_pb2 import Student, Course, EmailStudentTask


QUEUE = "EMAIL"

class EmailProducer():

        def __init__(self):
                self.connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
                self.channel = self.connection.channel()
                self.channel.queue_declare(queue=QUEUE, durable=True)
                return

        def publish(self, message):
                self.channel.basic_publish(
                        exchange='',
                        routing_key=QUEUE,
                        body=message.SerializeToString())
                return

if __name__ == '__main__':
        email_producer = EmailProducer();
        emails =['john@gmail.com', 'beth@gmail.com', 'alex@gmail.com']
        courses = ['Intro to Computing', 'Linear Algebra', 'History of Chairs']
        while True:
                email = random.choice(emails)
                course = random.choice(courses)
                task = EmailStudentTask(
                                student=Student(email=email),
                                course=Course(name=course))
                print '[x] publishing {} {}'.format(email, course)
                email_producer.publish(task)
                time.sleep(3)
```
EmailConsumer.java

```java
import com.rabbitmq.client.*;

import java.io.IOException;

public class EmailConsumer {

    private static final String QUEUE = "EMAIL";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        final Connection connection = factory.newConnection();
        final Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE, true, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        channel.basicQos(1);

        final Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("[x] Received");
                try {
                    doWork(Tutorial.EmailStudentTask.parseFrom(body));
                } finally {
                    System.out.println("[x] Done");
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        channel.basicConsume(QUEUE, false, consumer);
    }

    private static void doWork(Tutorial.EmailStudentTask task) {
        String message = String.format(
            "Hello %s, you have are now registered for %s",
            task.getStudent().getEmail(),
            task.getCourse().getName());
        System.out.println(message);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException _ignored) {
            Thread.currentThread().interrupt();
        }
    }
}

```

## Reflection

The above code works but it has one major limitation. **Our current implementation limits the contents of our emails.**. We can only send emails who's contents depend only on the Student and Course messages(the fields of the EmailStudentTask message). What if we wanted to maintain a pool of workers that are capable of performing any task thats assigned to them from the queue? To do that we would we need to "genericize" the message that encapsulates our unit of work. We could just have a general message which holds some bytes that hold
the data needed to perform some task. Then we can deserialize these bytes into a
proto and go from there. But we need to know which proto should be used to
deserialize the message. The task message would need some way to communicate this
to the consumer. We can accomplish this as follows:
```proto
message QueueMessage {

    enum MessageType {
        UNKNOWN = 0;
        EMAIL_STUDENT_TASK = 1;
    }

    MessageType type = 1;
    bytes data = 2;
}
```

The producer would use the MessageType enum to determine the correct proto to use
to deserialize the bytes stored in the data field.

## Refactor

To realize this new implementation we need to make a few changes to our application code. On the producer side we really dont need much work
```python
...
# Populate an EmailStudentTask proto like before.
task = tutorial_pb2.EmailStudentTask(
    Student(email=email), Course(name=course))

# Populate a QueueMessage with the task
message = QueueMessage(
    # Specify the type so the consumer can know how to decode it.
    type=QueueMessage.EMAIL_STUDENT_TASK
    data=task.SerializeToString())
print '[x] publishing {} {}'.format(email, course)
email_producer.publish(message)
```

At the consumer we need to use the MessageType enum to determine which proto to use to deserialize the message. This can be accomplished as follows:

```java
private static void doWork(Tutorial.QueueMessage message) throws Exception {
    switch(message.getType()) {
        case EMAIL_STUDENT_TASK:
            Tutorial.EmailStudentTask emailStudentTask =
                    Tutorial.EmailStudentTask.parseFrom(message.getData());
            String text = String.format(
                    "Hello %s, you have been registered for %s",
                    emailStudentTask.getStudent().getEmail(),
                    emailStudentTask.getCourse().getName());
            System.out.println(text);
            break;
    }

    try {
        Thread.sleep(1000);
    } catch (InterruptedException _ignored) {
        Thread.currentThread().interrupt();
    }
}
```

In order to add support for another task we have to:

 1. Create a new message for the task.
 2. Add an enum in our QueueMessage message.
 3. Deserialize the message using said enum.

## Conclusion

In this post we examined how to use RabbitMQ and Protocol Buffers in tandem. We began by solving a somewhat specific problem and concluded by generalizing our
implementation to be able to handle a wide variety of tasks. Granted we didn't actually implement the part of our scenario that involved sending emails. But it should be easy to see how we can use the generated protos to send and template emails. The code for this tutorial can be found at [this repository](https://github.com/SimplyFaisal/flexible_work_queues).


