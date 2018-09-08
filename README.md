### Back Pressure and its benifits

Event based Applications whether they belong to LAMBDA Architecture or KAPPA Architecture need to have an important feature of back pressure. The reason is simple. If your processing rate is slower than messages getting produced, and the application keep on consuming the events then application will have all those messages in memory. Due to which following problems can happen:

1. This may become a reason for the application to crash because the of memory exceeding from a limit. Which will eventually become the reason to loose all events which have been consumed
2. Even if application is stopped in that case also all the messages will be lost.

So it is very much important, application should be able to limit the message in take rate, so that messages can reside at message source instead of getting consumed. This is called Back Pressure.

### Implementation of Back Pressure with AKKA Actor and Kafka

![blog.jpg](blog.jpg)

Above diagram contains following Actors:

1. Ticker Actor
2. WorkerRouter Actor 
3. Worker Actor

### Ticker Actor 

This is the actor which uses Kafka High Level Consumer API. See below api to create a consumer using subscribe to a topic. Here the important point to be noted is, this way of creating consumer needs to mention consumer group.

```scala
object KafkaUtil {
  def createKafkaConsumer(properties: Properties): KafkaConsumer[String, String] = {
    val props = new Properties()
    props.put("bootstrap.servers", properties.getProperty(ConfigConstants.KAFKA_BOOT_SERVERS));
    props.put("group.id", properties.getProperty(ConfigConstants.KAFKA_GROUP_ID));
    props.put("auto.offset.reset", "latest");
    props.put("enable.auto.commit", "false");
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    val consumer = new KafkaConsumer[String, String](props)
    consumer.subscribe(properties.getProperty(ConfigConstants.KAFKA_TOPIC).split(",").toList.asJava)
    consumer
  }
  
  def createKafkaConsumer2(properties: Properties): KafkaConsumer[String, String] = {
    val props = new Properties()
    props.put("bootstrap.servers", properties.getProperty(ConfigConstants.KAFKA_BOOT_SERVERS));
    props.put("auto.offset.reset", "latest");
    props.put("enable.auto.commit", "false");
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    val consumer = new KafkaConsumer[String, String](props)
    consumer
  }  
}
```

It does not actually consumes the message but it finds out the current offset and latest offset of the corresponding KAFKA Topic. 

```scala
object Ticker {
  def props(properties: Properties, consumer: KafkaConsumer[String, String]) = Props(new Ticker(properties, consumer))
}

class Ticker(properties: Properties, consumer: KafkaConsumer[String, String]) extends Actor with Timers with ActorLogging {
	timers.startSingleTimer(TickKey, Tick, ConfigConstants.START_TICK_INTERVAL.seconds)
	protected var currentOffsets = Map[TopicPartition, Long]()

	override def preStart = {
    		val c = consumer
    		paranoidPoll(c)
    		if (currentOffsets.isEmpty) {
      			currentOffsets = c.assignment().asScala.map { tp =>
        			tp -> c.position(tp)
      			}.toMap
    		}
    		c.pause(this.currentOffsets.keySet.toList: _*)
  	}

    	override def receive = {
    		case Tick =>
      			consumeLimitedBatch()
      			timers.startPeriodicTimer(TickKey, Tick, ConfigConstants.TICK_INTERVAL.seconds)
  	}
}
```

At every TICK message which is sent to itself this Actor polls the Kafka Topic and does following:

1. Get the latest offset for each partion of a topic

```scala
  protected def latestOffsets(): Map[TopicPartition, Long] = {
    val c: Consumer[String, String] = consumer
    paranoidPoll(c)
    val parts = c.assignment().asScala

    // make sure new partitions are reflected in currentOffsets
    val newPartitions = parts.diff(currentOffsets.keySet)
    // position for new partitions determined by auto.offset.reset if no commit
    currentOffsets = currentOffsets ++ newPartitions.map(tp => tp -> c.position(tp)).toMap
    c.pause(newPartitions.toList: _*)
    c.seekToEnd(currentOffsets.keySet.toList: _*)
    parts.map(tp => tp -> c.position(tp)).toMap
  }
```

2. Clamps the offset to a max limit deifined for a partition. That means, we application must not consume more than what is defined as max limit per partition

```scala
  protected def clamp(
    offsets: Map[TopicPartition, Long]): Map[TopicPartition, Long] = {

    offsets.map(partitionOffset => {
      val uo = partitionOffset._2
      partitionOffset._1 -> Math.min(currentOffsets(partitionOffset._1) + ConfigConstants.MAX_MESSAGES_PER_PARTITION, uo)
    })
  }
```  

3. Prepares an OffsetRanges object and send it to WorkerRouter Actor

```scala
 def consumeLimitedBatch() = {
    val offsetRanges = new ListBuffer[OffsetRange]()
    val untilOffset = clamp(latestOffsets())
    untilOffset.map({
      case (tp, uo) =>
        consumer.commitSync(Collections.singletonMap(tp, new OffsetAndMetadata(uo)))
        offsetRanges += OffsetRange(tp, currentOffsets(tp), uo)
    })
    currentOffsets = untilOffset
    context.actorSelection("../workerRouter") ! OffsetRanges(offsetRanges.toList)
  }
```

Following snippet launches the Ticker Actor,

```scala
val consumer = KafkaUtil.createKafkaConsumer(properties)
val ticker = actorSystem.actorOf(Ticker.props(properties, consumer), "ticker")
```
### WorkerRouter Actor

As shown in the diagram, Ticker Actor gets the current offset and until offset for each partition and sends it to WorkerRouter actor.
