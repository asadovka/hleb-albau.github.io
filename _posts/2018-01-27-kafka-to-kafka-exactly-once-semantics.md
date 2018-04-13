---
layout: post
title:  "Kafka To Kafka Exactly Once Semantics"
date:   2018-01-27
excerpt: ""
keywords:
  - kafka
  - apache kafka
  - stream processing
  - exactly once
---

>  There are only two hard problems in distributed systems: 2. Exactly once delivery 1. Guaranteed order of messages 2.
> Exactly once delivery.


## General Possible Streaming Semantics

  If only we lived in an ideal world... network and disk failures, human errors, and even simple bugs, never occur in
  regular developer life. But... it's only a dream. Depending on how you react on components fails, there are three
  possible behaviour of your system:

1. **At most once semantics:** Is a lower messaging guarantee. Your message will be produced/processed at most once.
Your *consumer(subscriber)* pull a message, commit current offset, and then process it. There is a case, when during
processing message, *consumer* fails, and after restart it will process next queue item. From the *producer* side, you can
send async message to queue, which can be lost somewhere before storage, and commit current state.

2. **At least once semantics:** As a in previous case, you *consumer(subscriber)* pull a message, but commit state only after
processing message. There is case, when you already process message, but *consumer* fails, and queue offset is not
committed. After restart, *consumer* will process previous message once again. Otherside, *producer* can send sync
message with
waiting for broker(storage) acknowledgement. *Broker* receives and stores the message, but right after
that it fails. *Producer* after some configured timeout will retry to send message. That leads to the message being
written N times.

3. **Exactly once semantics:** Very strong guarantee. Each message will be produced/processed exactly once. All
failures, will be handled accordingly. In general, exactly once is probably impossible! But...


## What does it mean "Kafka-to-Kafka"

But... for some cases, we can archive it. Kafka-to-Kafka one of them. So what does it mean "Kafka-To-Kafka"? Yeah, it's
very simple! You read a message from Kafka topic, process it, and than save result back to another Kafka topic.
One important point here: during processing message, your service can query other services to obtain missing data,
but save entities **only** back to Kafka.


## Example scenario

Let's consider some example scenario, to show Kafka-To-Kafka exactly once semantics.
Suppose we have kafka topic called *"files-with-transactions"* containing urls. Our service should get next link, 
download file, parse it into list of transactions, and store that list in topic *"transactions"*.


## Show me the code

First of all we will define our consumers part. Let's start from the general topics messages(records) processor.

``` kotlin
//TopicsRecordsProcessor.kt
private val log = LoggerFactory.getLogger(TopicsRecordsProcessor::class.java)!!

abstract class TopicsRecordsProcessor<K, V>(private val topic: String) {

    abstract protected val consumer: KafkaConsumer<K, V>
    abstract fun processRecord(partition: TopicPartition, record: ConsumerRecord<K, V>)

    private val closed = AtomicBoolean(false)

    fun startProcessing() {
        try {
            consumer.subscribe(listOf(topic))
            while (!closed.get()) {
                readAndProcessRecords()
            }
        } catch (e: Exception) {
            log.error("consumer record processing error", e)
        } catch (e: WakeupException) {
            // Ignore exception if closing
            if (!closed.get()) throw e
        } finally {
            consumer.close()
        }
    }

    private fun readAndProcessRecords() {

        val records = consumer.poll(100)

        for (partition in records.partitions()) {
            val partitionRecords = records.records(partition)
            for (record in partitionRecords) {
                processRecord(partition, record)
            }
        }
    }

    open fun stopProcessing() {
        closed.set(true)
        consumer.wakeup()
    }
}
```


As you can see, we just wrap default kafka consumer. Next, we should setup consumer itself.
``` kotlin
//TransactionFilesProcess.kt
private val consumerGroup = "files-with-transactions-group-1"

private val consumerProperties = Properties().apply {
    put("bootstrap.servers","localhost")
    put("group.id", consumerGroup)
    put("isolation.level", "read_committed")
    put("enable.auto.commit", false)
    put("auto.offset.reset", "earliest")
}
```

Get attention to the two lines here:
1. `put("isolation.level", "read_committed")` used to process only committed messages to kafka. We will talk about
transaction a bit latter.
2. `put("enable.auto.commit", false)` offset management will be done by our process.
Same settings for kafka producer.


``` kotlin
//TransactionFilesProcess.kt
private val producerProperties = Properties().apply {
    put("bootstrap.servers", "localhost")
    put("group.id", "transactions-group-1")
    put("transactional.id", "transactions-group-1-transaction-id")
}
```


Be careful. If your service should scale at some point to N working nodes, than **transactional.id** option should be 
unique for every node. Now we are ready to define **TransactionFilesProcess.class**


``` kotlin
//TransactionFilesProcess.kt
class TransactionFilesProcess : TopicsRecordsProcess<String, String>("files-with-transactions") {

    override val consumer = KafkaConsumer<String, String>(
            consumerProperties, JsonDeserializer(String::class.java), JsonDeserializer(String::class.java)
    )

    private val producer = KafkaProducer<String, Transaction>(
            producerProperties, JsonSerializer<String>(), JsonSerializer<Transaction>()
    ).apply { initTransactions() }


    override fun processRecord(partition: TopicPartition, record: ConsumerRecord<String, String>) {

        val fileUrl = record.value()
        val transactions = downloadAndParseFile(fileUrl)

        val offset = mapOf(partition to OffsetAndMetadata(record.offset() + 1))
        producer.beginTransaction()
        try {
            transactions.foreach { transaction -> producer.send(ProducerRecord(fileUrl, transaction))}
            producer.sendOffsetsToTransaction(offset, consumerGroup)
            producer.commitTransaction()
        } catch (e: Exception) {
            producer.abortTransaction()
            stopProcessing()
        }
    }

    override fun stopProcessing() {
        super.stopProcessing()
        producer.close()
    }
}
```


Listing below is straightforward. We download one file by url, parse it into list of transactions. Create offset 
object for latter committing consumer position. Kafka transactions are started by producers. To enable transaction 
support, producer should call `initTransactions()` method. Transaction allows you to send records and commit 
consumer offset atomically. 

The send is asynchronous and this method will return immediately once the record has been stored in the buffer of 
records waiting to be sent. This allows sending many records in parallel without blocking to wait for the response 
after each one. When used as part of a transaction, it is not necessary to define a callback or check the result of 
the future in order to detect errors from `send()`. If any of the send calls failed with an irrecoverable error, the 
final `commitTransaction()` call will fail and throw the exception from the last failed send. When this happens, your 
application should call `abortTransaction()` to reset the state and try once again to process record. Also, previously
we define consumer option `put("isolation.level", "read_committed")` to process only messages from committed 
transactions, thus we avoid processing records N times (from failed and completed transactions). 


## Final words.

While it's definitely easy to implement exactly-once semantic for kafka-to-kafka case, there are some caveats. You 
should be totally sure, that incoming(consumer) topics will not contain duplicated items itself or process
items idempotently(but it's not possible for many cases, ex: transaction processing with updating address balances).