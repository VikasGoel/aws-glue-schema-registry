
# AWS Glue Schema Registry Library
![Build Status](https://github.com/awslabs/aws-glue-schema-registry/workflows/MavenCI/badge.svg?branch=master&event=push)
[![Apache 2 License](https://img.shields.io/github/license/awslabs/s2n.svg)](http://aws.amazon.com/apache-2-0/)
![Java](https://img.shields.io/badge/language-java-blue.svg)

**AWS Glue Schema Registry** provides a solution for customers to centrally discover, control and evolve 
schemas while ensuring data produced was validated by registered schemas.
**AWS Glue Schema Registry Library** offers Serializers and Deserializers that plug-in with Glue Schema Registry.

## Getting Started
1. **Sign up for AWS** &mdash; Before you begin, you need an AWS account. For more information about creating an AWS 
account and retrieving your AWS credentials, see [AWS Account and Credentials](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/home.html) in the AWS SDK for Java Developer Guide.
1. **Sign up for AWS Glue Schema Registry** &mdash; Go to the AWS Glue Schema Registry console to sign up for the 
service and create an AWS Glue Schema Registry. For more information, see [Getting Started with Glue Schema 
Registry](https://docs.aws.amazon.com/glue/latest/dg/schema-registry-gs.html) in the AWS Glue Developer Guide.
1. **Minimum requirements** &mdash; To use the AWS Glue Schema Registry, you'll need **Java 1.8+**.

## Features

1. AVRO Messages/records are serialized on producer front and deserialized on the consumer front by using 
schema-registry-serde.
1. Kafka Streams support for AWS Glue Schema Registry.
1. Records can be compressed to reduce message size.
1. An inbuilt local in-memory cache to save calls to AWS Glue Schema Registry. The schema version id for a schema 
definition is cached on Producer side and schema for a schema version id is cached on the Consumer side.
1. Auto registration of schema can be enabled for any new schema to be auto-registered.
1. For AVRO Schemas, Evolution check is performed while registering.
1. Migration from a third party Schema Registry.
1. Flink support for AWS Glue Schema Registry.
1. Kafka Connect support for AWS Glue Schema Registry.

## Building from Source

After you've downloaded the code from GitHub, you can build it using Maven.

The following maven command will clean the target directory, compile the project, execute the tests and package the project build into a JAR.

`mvn clean install`

Alternatively, one could git clone this repo and run ``mvn clean install``.

## Testing

To simply run the tests, execute the following maven command: 

`mvn test`

## Using the AWS Glue Schema Registry Library Serializer / Deserializer
The recommended way to use the AWS Glue Schema Registry Library for Java is to consume it from Maven.

**Using AWS Glue Schema Registry with Amazon MSK** &mdash; To set-up Amazon Managed Streaming for Apache Kafka see 
[Getting started with Amazon MSK.](https://docs.aws.amazon.com/msk/latest/developerguide/getting-started.html)

### Maven Dependency
  ``` xml
  <dependency>
      <groupId>software.amazon.glue</groupId>
      <artifactId>schema-registry-serde</artifactId>
      <version>1.0.0</version>
  </dependency>
  ```
### Code Example

#### Producer for Kafka with AVRO format

```java
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, AWSKafkaAvroSerializer.class.getName());
        properties.put(AWSSchemaRegistryConstants.AWS_REGION, "us-east-1");
        properties.put(AWSSchemaRegistryConstants.REGISTRY_NAME, "my-registry");
        properties.put(AWSSchemaRegistryConstants.SCHEMA_NAME, "my-schema");

        Schema schema_payment = null;
        try {
            schema_payment = parser.parse(new File("src/main/resources/avro/com/tutorial/Payment.avsc"));
        } catch (IOException e) {
            e.printStackTrace();
        }
        
        GenericRecord musical = new GenericData.Record(schema_payment);
        musical.put("id", "entertainment_2");
        musical.put("amount", 105.0);

        List<GenericRecord> misc = new ArrayList<>();
        misc.add(musical);

        try (KafkaProducer<String, GenericRecord> producer = new KafkaProducer<String, GenericRecord>(properties)) {
            for (int i = 0; i < 4; i++) {
                GenericRecord r = misc.get(i);

                final ProducerRecord<String, GenericRecord> record;
                record = new ProducerRecord<String, GenericRecord>(topic, r.get("id").toString(), r);

                producer.send(record);
                System.out.println("Sent message " + i);
                Thread.sleep(1000L);
            }
            producer.flush();
            System.out.println("Successfully produced 10 messages to a topic called " + topic);

        } catch (final InterruptedException | SerializationException e) {
            e.printStackTrace();
        }
```

#### Consumer for Kafka with AVRO format

```java
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, AWSKafkaAvroDeSerializer.class.getName();
        properties.put(AWSSchemaRegistryConstants.AWS_REGION, "us-east-1");
        properties.put(AWSSchemaRegistryConstants.AVRO_RECORD_TYPE, AvroRecordType.GENERIC_RECORD.getName());
        
        try (final KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<String, GenericRecord>(properties)) {
            consumer.subscribe(Collections.singletonList(topic));

            while (true) {
                final ConsumerRecords<String, GenericRecord> records = consumer.poll(100);
                for (final ConsumerRecord<String, GenericRecord> record : records) {
                    final String key = record.key();
                    final GenericRecord value = record.value();
                    System.out.println("Received message: key = " + key + ", value = " + value);
                }
            }
        }

```

### Using AWS Glue Schema Registry with Kinesis Data Streams

**Kinesis Client library (KCL) / Kinesis Producer Library (KPL):** [Getting started with AWS Glue Schema Registry with AWS Kinesis Data Streams](https://docs.aws.amazon.com/glue/latest/dg/schema-registry-integrations.html#schema-registry-integrations-kds)

If you cannot use KCL / KPL libraries for Kinesis Data Streams integration,
**See [examples](examples/) for working example 


### Using Auto-Registration

Auto-Registration allows any record produced with new schema to be automatically registered with the AWS Glue Schema 
Registry. The Schema is registered automatically and a new schema version is created and evolution checks are performed.

If the Schema already exists, but the schema version is new, the new schema version is created and evolution checks are performed. 

Auto-Registration is disabled by default. To enable Auto-Registration, enable setting by passing the configuration to
 the Producer as below :

```java
    properties.put(AWSSchemaRegistryConstants.SCHEMA_AUTO_REGISTRATION_SETTING, true); // If not passed, defaults to false
```

### Providing Registry Name

Registry Name can be provided by setting this property - 

```java
    properties.put(AWSSchemaRegistryConstants.REGISTRY_NAME, "my-registry"); // If not passed, uses "default-registry"
```

### Providing Schema Name

Schema Name can be provided by setting this property - 

```java
    properties.put(AWSSchemaRegistryConstants.SCHEMA_NAME, "my-schema"); // If not passed, uses transport name (topic name in case of Kafka)
```

### Providing Registry Description

Registry Description can be provided by setting this property - 

```java
    properties.put(AWSSchemaRegistryConstants.DESCRIPTION, "This registry is used for several purposes."); // If not passed, constructs a description
```

### Providing Compatibility Setting for Schema

Registry Description can be provided by setting this property - 

```java
    properties.put(AWSSchemaRegistryConstants.COMPATIBILITY_SETTING, Compatibility.FULL); // Pass a compatibility mode. If not passed, uses Compatibility.BACKWARD
```

### Using Compression

Deserialized byte array can be compressed to save on data usage over the network and storage on the topic/stream. The 
Consumer side using AWS Glue Schema Registry Deserializer would be able to decompress and deserialize the byte array.
By default, compression is disabled. Customers can choose ZLIB as compressionType by setting up below property.

```java
    // If not passed, defaults to no compression
    properties.put(AWSSchemaRegistryConstants.COMPRESSION_TYPE, AWSSchemaRegistryConstants.COMPRESSION.ZLIB.name());
```

### In-Memory Cache settings

In Memory cache is used by Producer to store schema to schema version id mapping and by consumer to store schema 
version id to schema mapping. This cache allows Producers and Consumers to save time and hits on IO calls to Schema 
Registry.

The cache is available by default. However, it can be fine-tuned by providing cache specific properties.

```java
    properties.put(AWSSchemaRegistryConstants.CACHE_TIME_TO_LIVE_MILLIS, "60000"); // If not passed, defaults to 24 Hours
    properties.put(AWSSchemaRegistryConstants.CACHE_SIZE, "100"); // Maximum number of elements in a cache - If not passed, defaults to 200
```

### Migrating from a third party Schema Registry

To migrate to AWS Glue Schema Registry from a third party schema registry for AVRO data types for Kafka, add this 
property for value class along with the third party jar.

```java
    properties.put(AWSSchemaRegistryConstants.SECONDARY_DESERAILIZER, <ThirdPartyKafkaDeserializer>);
```

### Using Kafka Connect with AWS Glue Schema Registry

* Clone this repo, build and copy dependencies

```java
git clone git@github.com:awslabs/aws-glue-schema-registry.git
cd aws-glue-schema-registry
mvn clean install
mvn dependency:copy-dependencies
```

* Configure Kafka Connectors with following properties

```java
    key.converter=com.amazonaws.services.schemaregistry.kafkaconnect.AWSKafkaAvroConverter
    value.converter=com.amazonaws.services.schemaregistry.kafkaconnect.AWSKafkaAvroConverter
    key.converter.region=ca-central-1
    value.converter.region=ca-central-1
    key.converter.schemaAutoRegistrationEnabled=true
    value.converter.schemaAutoRegistrationEnabled=true
    key.converter.avroRecordType=GENERIC_RECORD
    value.converter.avroRecordType=GENERIC_RECORD
```
* Add command below to *Launch mode* section under *kafka-run-class.sh*

``` 
-cp $CLASSPATH:"<your aws glue schema registry base directory>/target/dependency/*" 
```

It should look like this

```
    # Launch mode
    if [ "x$DAEMON_MODE" = "xtrue" ]; then
      nohup "$JAVA" $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS -cp $CLASSPATH:"/Users/johndoe/aws-glue-schema-registry/target/dependency/*" $KAFKA_OPTS "$@" > "$CONSOLE_OUTPUT_FILE" 2>&1 < /dev/null &
    else
      exec "$JAVA" $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS -cp $CLASSPATH:"/Users/johndoe/aws-glue-schema-registry/target/dependency/*" $KAFKA_OPTS "$@"
    fi
```

* If using bash, run the below commands to set-up your CLASSPATH in your bash_profile. (For any other shell, update the environment accordingly.)
  ```bash
      echo 'export GSR_LIB_BASE_DIR=<>' >>~/.bash_profile
      echo 'export GSR_LIB_VERSION=1.0.0' >>~/.bash_profile
      echo 'export KAFKA_HOME=<your kafka installation directory>' >>~/.bash_profile
      echo 'export CLASSPATH=$CLASSPATH:$GSR_LIB_BASE_DIR/avro-kafkaconnect-converter/target/schema-registry-kafkaconnect-converter-$GSR_LIB_VERSION.jar:$GSR_LIB_BASE_DIR/common/target/schema-registry-common-$GSR_LIB_VERSION.jar:$GSR_LIB_BASE_DIR/avro-serializer-deserializer/target/schema-registry-serde-$GSR_LIB_VERSION.jar' >>~/.bash_profile
      source ~/.bash_profile
    ```
* (Optional) If you wish to test with a simple file source then clone the file source connector.
  
  ```bash
      git clone https://github.com/mmolimar/kafka-connect-fs.git
      cd kafka-connect-fs/
    ```
  
  Under source connector configuration(config/kafka-connect-fs.properties), edit the data format to Avro, file reader 
  to AvroFileReader and update an 
  example Avro object from the file path you are reading from. For example:
  
  ```
      fs.uris=<path to a sample avro object>
      policy.regexp=^.*\.avro$
      file_reader.class=com.github.mmolimar.kafka.connect.fs.file.reader.AvroFileReader
  ```
  
  Install source connector
  
  ```
      mvn clean package
      echo "export CLASSPATH=\$CLASSPATH:\"\$(find target/ -type f -name '*.jar'| grep '\-package' | tr '\n' ':')\"" >>~/.bash_profile
      source ~/.bash_profile
  ```
  
  Update the sink properties under *<your Apache Kafka installation directory>/config/connect-file-sink.properties*
  
  ```
  file=<output file full path>
  topics=<my topic>
  ```
  
  Start Source Connector (In this example it is file source connector)
  
  ```
  $KAFKA_HOME/bin/connect-standalone.sh $KAFKA_HOME/config/connect-standalone.properties config/kafka-connect-fs.properties
  ```
 
  Run Sink Connector (In this example it is file sink connector))
  
  ```
  $KAFKA_HOME/bin/connect-standalone.sh $KAFKA_HOME/config/connect-standalone.properties $KAFKA_HOME/config/connect-file-sink.properties
  ```

### Using Kafka Streams with AWS Glue Schema Registry

### Maven Dependency
  ``` xml
  <dependency>
        <groupId>software.amazon.glue</groupId>
        <artifactId>schema-registry-kafkastreams-serde</artifactId>
        <version>1.0.0</version>
  </dependency>
  ```

```java
    final Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "avro-streams");
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 0);
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, AWSKafkaAvroSerDe.class.getName());
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
    
    props.put(AWSSchemaRegistryConstants.AWS_REGION, "us-east-1");
    props.put(AWSSchemaRegistryConstants.SCHEMA_AUTO_REGISTRATION_SETTING, true);
    props.put(AWSSchemaRegistryConstants.AVRO_RECORD_TYPE, AvroRecordType.GENERIC_RECORD.getName());

    StreamsBuilder builder = new StreamsBuilder();
    final KStream<String, GenericRecord> source = builder.stream("avro-input");
    final KStream<String, GenericRecord> result = source
        .filter((key, value) -> !"pink".equals(String.valueOf(value.get("favorite_color"))));
        .filter((key, value) -> !"15.0".equals(String.valueOf(value.get("amount"))));
    result.to("avro-output");

    KafkaStreams streams = new KafkaStreams(builder.build(), props);
    streams.start();
```

## Using the AWS Glue Schema Registry Flink Connector
The recommended way to use the AWS Glue Schema Registry Flink Connector for Java is to consume it from Maven.

**Minimum requirements** &mdash; Apache Flink versions supported **Flink 1.11+**

**Working with Kinesis Data Analytics** &mdash; AWS Glue Schema Registry can be setup with [Amazon Kinesis Data 
Analytics 
for Apache Flink](https://docs.aws.amazon.com/kinesisanalytics/latest/java/what-is.html).

For using Amazon VPC with Kinesis Data Analytics please see [Configuring Kinesis Data Analytics for Apache Flink 
inside Amazon VPC.](https://docs.aws.amazon.com/kinesisanalytics/latest/java/vpc.html) 

### Maven Dependency
  ``` xml
  <dependency>
       <groupId>software.amazon.glue</groupId>
       <artifactId>schema-registry-flink-serde</artifactId>
       <version>1.0.0</version>
  </dependency>
  ```
### Code Example

#### Flink Kafka Producer with AVRO format

```java
    String topic = "topic";
    Properties properties = new Properties();
    properties.setProperty("bootstrap.servers", "localhost:9092");
    properties.setProperty("group.id", "test");

    Map<String, Object> configs = new HashMap<>();
    configs.put(AWSSchemaRegistryConstants.AWS_REGION, "us-east-1");
    configs.put(AWSSchemaRegistryConstants.SCHEMA_AUTO_REGISTRATION_SETTING, true);

    Schema.Parser parser = new Schema.Parser();
    Schema schema = parser.parse(new File("path/to/avro/file"));

    FlinkKafkaProducer<GenericRecord> producer = new FlinkKafkaProducer<>(
            topic,
            GlueSchemaRegistryAvroSerializationSchema.forGeneric(schema, topic, configs),
            properties);
    stream.addSink(producer);
```

#### Flink Kafka Consumer with AVRO format

```java
    String topic = "topic";
    Properties properties = new Properties();
    properties.setProperty("bootstrap.servers", "localhost:9092");
    properties.setProperty("group.id", "test");

    Map<String, Object> configs = new HashMap<>();
    configs.put(AWSSchemaRegistryConstants.AWS_REGION, "us-east-1");
    configs.put(AWSSchemaRegistryConstants.AVRO_RECORD_TYPE, AvroRecordType.GENERIC_RECORD.getName());

    Schema.Parser parser = new Schema.Parser();
    Schema schema = parser.parse(new File("path/to/avro/file"));

    FlinkKafkaConsumer<GenericRecord> consumer = new FlinkKafkaConsumer<>(
            topic,
            GlueSchemaRegistryAvroDeserializationSchema.forGeneric(schema, configs),
            properties);
    DataStream<GenericRecord> stream = env.addSource(consumer);
```
