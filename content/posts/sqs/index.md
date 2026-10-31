---
title: "A deep-dive into SQS queue types and their guarantees"
# author: "Harshit Tiwari"
# authorAvatarPath: "/avatar.jpeg"
date: "2026-10-31"
summary: "Deep-diving into the two types of SQS queues and the guarantees each one provides"
description: "Deep-diving into the two types of SQS queues and the guarantees each one provides"
toc: true
readTime: true
autonumber: true
math: true
tags: ["sqs", "message-queue"]
showTags: false
---

There are two types of queues that AWS's SQS provides: Standard and FIFO.

Before choosing which one to use, it is very important that we understand basic differences between the both, which guarantees each one provides and even more importantly what those guarantees mean in the first place.

## Standard queue

### Best effort FIFO

A standard queue only ensures "best effort FIFO". This means that SQS _attempts_ to deliver messages in the order they arrived, but does not make a guarantee to do so. Under normal conditions, the order will be preserved, but SQS will not hold back from sacrificing the order if any conflict occurs (like a network delay or a failure).

The reason for this is that a standard queue is built over a highly distributed system and is designed to optimize for throughput than for accuracy or order preservation.

Therefore, it is very much possible that the message delivery might be out of order and your system should never rely on a standard queue if strict FIFO processing is critical to your application's business logic.

### At-least-once delivery

SQS only ensures at-least-once delivery of messages, which means a message can be sent more than once. There are two scenarios how this can happen:

Duplicate message is sent by the producer. Suppose a network glitch occurs right after your producer sends a message, your producer might think the send failed and try again.
SQS does not prevent duplicate messages to be enqueued to a standard queue. Since duplicate messages are present in the queue, they'll be delivered individually.

Another way more than once delivery could happen is due to the highly distributed architecture of AWS. SQS stores copies of the messages across multiple servers to enable redundancy and high availability.

Let's say a message X is stored on three servers: 1, 2 and 3.

A consumer "A" requests for a message and Server 1 returns the message X. Now server 1 marks the message invisible within its server in order to respect X's visibility timeout. But server 1 also needs to inform server 2 and 3 which hold a copy of this message to mark this message as invisible in their servers as well.
This communication from server 1 to 2 and 3 could fail for multiple reasons. Server 2/3 could be unavailable during that time or there could be a network delay. Because of this the other servers might not be aware of the in-flight state of the message X.
If another consumer B requests a message from server 2 before the communication could reach server 2, it will handover the same message to consumer B, thus delivering the same message twice.
This might not happen always and only on rare occasions, but your system should be fool proof to handle such scenarios.

Therefore, the consumers polling from a standard queue should always be idempotent in order to prevent unintended consequences due to duplicate operations.

## FIFO queue

### Strict FIFO

In contrast to a standard queue, a FIFO queue ensures "strict FIFO" delivery of messages. It does so using something known as a "message group".
A message group is just a way to organise messages in a queue in distinct, ordered subsets.
Every message sent to a FIFO queue _should_ have a `MessageGroupId` attribute, or the request will fail.

SQS guarantees _strict_ FIFO ordering within a message group, i.e. all the messages with the same `MessageGroupId` will be delivered in a strict FIFO order.

How does it do that?

If a message from a message group is currently in-flight, the next message from the same message group won't be delivered until the previous message is deleted from the queue. This is what makes a FIFO queue different from a standard queue.

Let's see an example:

Suppose a consumer polls a batch of 10 messages from the queue. SQS will ensure all these messages are from same message group, unless of course if all the messages in that message group are in-flight in which case it will send message(s) from another message group.

The most important thing to note here is that : the next batch of 10 messages (from the same message group) won't be delivered until the previous batch has been deleted. If the visibility timeout of the batch expires before the batch could be deleted, the batch will appear in the same place in the queue again (that is at the front) and will be polled again by the consumer. This is how strict FIFO ordering is maintained within a message group.

However, messages in different message groups are independent and can be delivered parallelly. Therefore we can use multiple consumers to poll messages from multiple message groups.

Suppose there are two consumers: 1 and 2 polling from the queue. Also, say, the messages in the queue are organised in three message groups: A, B and C - A has 9 messages, B has 10 messages and C has 11 messages.
Consumer 1 requested for a batch of 10 messages and say SQS delivered 9 messages from message group A. It has to send one more message to fill the batch, so it sends 1 message from message group B.

Now while consumer 1 is processing these 10 messages, consumer 2 also requests for a batch of 10 messages. SQS will find the oldest message from a message group in which no other message is currently in-flight. There are 9 messages in-flight from A and 1 message in-flight from B. The only message group from which currently there are no messages in-flight is C. So it will send 10 messages from message group C.

This [AWS blog](https://aws.amazon.com/blogs/compute/solving-complex-ordering-challenges-with-amazon-sqs-fifo-queues/) summarises the rules of message delivery in a FIFO queue very well:

> The order of messages leaving a FIFO queue is governed by three rules:
>
> 1. Return the oldest message where no other message in the same message group is currently in-flight.
> 2. Return as many messages from the same message group as possible.
> 3. If a message batch is still not full, go back to rule 1.

Both consumer 1 and consumer 2 process the messages they received parallelly.

Therefore, by using multiple message groups we can maintain high throughput while still ensuring strict FIFO order within a message group.

However, note that the throughput in a FIFO queue will not be as high as a standard queue. Even a single in-flight message from a message group, blocks that entire message group in a FIFO queue. A standard queue, on the other hand, doesn't wait for a message to be deleted to deliver the next message. Therefore, there's no reason to use a FIFO queue over a standard queue unless strict FIFO ordering matters for your application.

### Exactly-once delivery?

There's a reason why I put a question mark above.
In contrast to a standard queue, a FIFO queue is said to ensure "exactly once" delivery of messages. Because this is a big claim, we'll take it with a pinch of salt and also see several nuances associated with it.

We discussed two ways a message might be delivered more than once in a standard queue.
To go from at-least-once delivery to exactly-once, SQS needs to solve those two problems:

- Case 1: The same message is enqueued twice by the producer and is therefore delivered twice.
- Case 2: Even if the message was enqueued once, it might still be delivered twice to different consumers due to the quirks of distributed systems

The first problem is solved using: Message deduplication Id.

Each message in a FIFO queue must have a unique `MessageDeduplicationId`. You can either specify this id explicitly while sending the message, or you can enable content-based deduplication in which case SQS generates a SHA-256 hash of the message body and uses that as the deduplication id.

If a message is sent with a id that matches the id of message that was already enqueued, SQS silently rejects that message. For example, due to a network failure, a producer might think that the enqueue failed and might retry to enqueue the same message (i.e. same id) which was already sent. SQS will prevent this message from entering the queue.

There's one constraint to be aware of: SQS only tracks the `MessageDeduplicationId` within a rolling 5-minute window. If a message with same id is sent at 6th minute, SQS will consider it to be unique and enqueue it successfully. If your application runs the risk of generating duplicate messages over a period longer than 5 minutes (e.g., hours or days apart), you cannot rely solely on SQS FIFO deduplication.

So the problem 1 of duplicate enqueues on the producer side is solved although with a caveat: AWS does prevent a duplicate message to enter the queue as long as the producer does not enqueue the same message 5 minutes apart.

But on the consumer side, how does FIFO queue prevent duplicate delivery of the same message that might happen due to distributed server lag? The real honest answer to this question is: we don't know. AWS doesn't publish its internals and our only option is to believe what is said in the documentation which says that a message in a FIFO queue will never be delivered twice while it is in-flight. The only explanation the docs give is for the `MessageDeduplicationId` but that doesn't explain how it ensures to synchronize the in-flight status of a message across every server holding a copy of that message.

Let's say we trust AWS's claim on this. Still, it is important to understand the extent of this "exactly-once" guarantee.

The "exactly-once" guarantee which FIFO queues claim to give only holds _during_ the visibility timeout window.
This is a summary of how the two types of queue will behave _before_ the visibility timeout has expired:

- Standard: A message once delivered, is usually hidden, however due to distributed server lag or duplicate enqueue by producer, it might get delivered multiple times
- FIFO: A message once delivered is _guaranteed_ not to be delivered to anyone else, while the message is in-flight. How exactly? We don't know, but its a guarantee which AWS claims to give and we've to take their word for it.

However, consider these scenarios:

- The consumer crashes mid processing, or takes longer than the visibility timeout to process the message
- The consumer makes the delete call but the delete call fails due to a network issue

In these scenarios, the message is not deleted within the visibility timeout window and _will_ be delivered again to the same or some other consumer (and thus duplicate processing), regardless of which type of queue you're using. Therefore, your consumer should _always_ be idempotent to prevent unintended consequences, no matter which queue you use.

It is probably because of such scenarios, that most references including the official docs, call this guarantee "exactly-once processing" and not exactly-once delivery (as re-deliveries are possible). I am not a fan of this terminology, but what they actually mean by exactly-once processing is that FIFO queues prevent duplicates to be sent to the consumer from the producer side (that too within the 5 minute limit).

More precisely a FIFO queue ensures an actual exact-once processing only if: producers do not enqueue duplicate messages wider than 5 minutes apart AND consumers never fail to delete the messages they have processed within the visibility timeout period. Those are two big IFs especially in a high throughput distributed environment.

But we should also acknowledge that it is not SQS's responsibility to make the system using its queues completely consistent. Basically as long as the message is in control of the queue, FIFO queue does give exactly-once guarantee, but it cannot do anything about the failures on producer side (duplicate enqueues) or the consumer side (failure to delete messages).

FIFO queues prioritize accuracy and order while standard queues prioritize throughput and availability. Unless strict FIFO ordering matters for your application, you'll always be better off using a standard queue over a FIFO queue.
