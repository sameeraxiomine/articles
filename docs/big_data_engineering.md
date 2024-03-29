
The goal of next generation open-source Big Data frameworks such as Spark and Flink is to match the performance and user-friendly expressiveness of Massively Parallel Processing (MPP) databases such as Netezza. SQL interfaces such as Hive and SparkSQL provide the expressiveness. However, matching the performance is proving to be a much harder hurdle to cross. 

Since all major Big Data Frameworks (Hadoop, Spark, Flink) use the JVM, they are subject to two JVM specific constraints-

1. Java objects consume considerably more memory than the byte representation of the attributes they contain. 

2. Major Garbage Collection pauses will thwart your performance goals when you attempt to cache too much data (java objects) for longer duration's in memory for in-memory processing

All the engineering techniques employed by Flink, Spark, Hadoop and HBase to achieve high performance (levels of performance you might expect when you program directly in C) aim to minimize the impact of the above two constraints. This article will explore these techniques. 

All performance improvements aside, the following blog [entry](http://0x0fff.com/spark-dataframes-are-faster-arent-they/ "entry") by Alexey Grishchenko should serve as a sober reminder that while the performance of Java based Big Data frameworks is vastly improving, it will be long before they can catch up with MPP system performance levels. 


## 80/20 Rule for performance ##

Not all operations are equal. A small percentage of operations have high performance overheard. Most Big Data frameworks work with the following combination of operations-

1. **Compressing vs. Un-Compressing** - This directly affects how fast data is read or written out. CPU consumption varies across compression schemes.

2. **Sorting** - Big Data processing depends on the "Reduce" operation. Sorting is the key operation in performing the "Reduce" phase in Map-Reduce. Since the "Join" operation depends on "Reduce", "Sorting" key to the "Join" operation.    

3. **Shuffle** - The second most important operation for the "Reduce" operation is "Shuffle". Consequently the "Shuffle" operation is also key to performing the "Join" operation.

4. **Join** - "Map-only" jobs normally operate at high performance. Performance usually becomes a factor only when you have "Reduce" operation in your jobs. A "Reduce" operation implies a join. It is possible to perform joins in map-only jobs using broadcast joins. Broadcast joins can be used when one side of the join is small enough to be held in memory. The smaller dataset is broadcast to all nodes and maintained in a HashMap indexed by join key. The larger dataset streams in and joins with the smaller dataset by looking up the record by the join key from the in-memory Hashmap. Map-side joins can even be performed when both datasets are large if both datasets are sorted by the join key. Pig utilizes this technique, known as the merge-joins where both (join key sorted) datasets are traversed in lock-step and joined in the Map phase itself (there is a pre-indexing step as well. See this [link](http://pig.apache.org/docs/r0.10.0/perf.html#merge-joins) for more details). However, it is rare for datasets to be naturally sorted by the join key. "Reduce" operation is usually needed when joining large datasets. 

The key to making Big Data processing fast is to address the performance of the above operations. 

### Performance Constraints and JVM Characteristics ###

As discussed earlier, Java imposes two primary constraints on performance

1. Java Objects have considerable storage overhead. 
2. Utilizing large number of small long-lived objects makes the performance throughput suffer due to large amounts of time spent in Major Garbage Collection

Performing big-data operations by only operating on the JVM heap can easily lead to OutOfMemoryError's. This will kill the JVM (Unlike an Exception, Error cannot be recovered from). This can happen for a variety of reasons in even the most innocuous of Big Data applications. For example, a skew in data which has a large number of elements for a few key values could cause a join on that key attribute to fail for a reduce operations on that key if all the elements of that key are kept in memory. Techniques like "Secondary-Sorting" are utilized to avoid having to keep all elements for a reduce-side key in memory. Graph applications are an example of applications which belong to this category. 

This is the primary reason why Big Data frameworks use memory judiciously and employ techniques like disk-spills. This carries a performance penalty but is necessary to ensure that the framework remains robust when subjected to all kinds of load. For example, persisting data to disk is a technique used in the Pig framework. The implementations of the `DataBag` interface, used extensively in Pig, support spilling to disk when the number of tuples reaches a pre-defined limit (default is # `20000`). 


#### JVM Storage Overhead ####

A typical object stored in the Java heap consists of-

1. **Object Header** - Few bytes are used for housekeeping information. The Hotspot VM utilizes 8 bytes for the header for a normal object and an additional 4 bytes for an array object to store the size of the array 
2. **Primitive Fields** - Bytes required to store primitive fields. For example, 1 byte is used to store boolean, 2 for char, 4 for int ((See [12](http://www.javamex.com/tutorials/memory/object_memory_usage.shtml))
3. **Reference Storage** - Memory for reference fields (4 bytes each)
4. **Padding for rounding off** - padding to round of total bytes storage per object to be in multiple of 8

As shown in figure 1, an instance of a Java class which contains an instance variable of the primitive type boolean will occupy 8 bytes for the header, 1 byte for the boolean instance and 7 bytes for rounding off. However a single bit (1/8th of a byte) would have sufficed. 
![Silvrback blog image](https://silvrback.s3.amazonaws.com/uploads/dd3bd299-2713-4ffe-9b94-4e3913a946c8/JVMObjectOverhead_large.png)

*Figure 1: Memory footprint of a Java class with a single boolean instance variable* 

This example demonstrates the tremendous storage overhead for a typical Java object. The storage overhead becomes substantial when working with instances of complex classes like Hashmap and List. 


##### JVM Storage Overhead impacts Serialization/Deserialization #####

At Big Data scale you constantly need to serialize and de-serialize objects. This occurs when -

1. Mappers write output to local disk.
2. Mappers read output from local disk for Map side sorting
3. When Reducers pull sorted/shuffled output from Mapper node

In each of the above instances, you are working with considerably more data than represented by the raw byte representation.

Storage overhead of Java objects impacts how efficiently the memory hierarchy is utilized. Even if all the data you need is in memory, it is unlikely that L1/L2/L3 caches will be large enough to hold it. This leads to excessive cache evictions slowing performance

##### JVM Storage Overhead impacts Garbage Collection #####

The key idea behind Java Garbage Collection is, most object die young. The Java heap memory (JVM Managed Memory) is divided broadly into two regions, young generation and tenured generation. Typically the young generation comprises of 20% of the total managed memory space and the remaining 80% goes to the tenured generation. Objects are allocated to the young generation when they are first created. When the young generation fills up, the minor collection executes. During this phase objects which are no longer referenced are cleared and those still referenced are moved to the tenured generation. Eventually the Major Collection runs when the tenured collection fills up. Major collection will traverse every object in the graph (also known as the sweeping phase) and reclaim memory from the tenured generation. Note that this is a very simplified explanation of Garbage Collection. A detailed and graphic description of Garbage Collection can be found [here](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html).

JVM memory overhead for storing objects reduces the threshold for OutOfMemoryError's. There is considerably less memory available to store objects than you would calculate based on the storage need to accommodate the raw bytes of the primitives variables stored in the java instances. 

Big Data frameworks are moving towards in-memory storage and operations over time. Map-Reduce was primarily I/O bound. Early implementations of Iterative algorithms using MapReduce (Ex. Mahout) require each iteration to read the results of the prior iteration from disk (HDFS) and write out the results of the current iteration back to disk (HDFS). Combined with the overhead of starting MR jobs, this approach was very slow.

Spark came along with the concept of RDD's where data and can be cached in a distributed manner in memory between iterations. This removes the excessive I/O cost of writing-to/reading-from disk as also the overhead for starting/stopping jobs. However, the tradeoff is there will be risk of putting strain on the Garbage Collection processes. The reason being, while Spark employs disk-spilling to manage excessive use of memory for the RDD's, there is that much less memory available to the user program to utilize for custom objects. 


##### Evolution of Serialization Schemes in Java #####

Java Serialization schemes have evolved over time. The default `java.io.Serializable` is expensive since every instance stores the class metadata as well as the meta-data for the types referenced from within the class. This scheme is ideal when you want the serialized byte representation to contain all the information it needs to de-serialize back to a Java instance. 

The `java.io.Externalizable` interface evolved to handle part of the problem. When using this interface, the programmer controls how the attributes of the class are serialized and de-serialized. Hence the meta-data for the class attributes is no longer need to be stored in the serialized byte stream. However the class meta-data of the serialized instance is stored in the serialized byte stream since the `java.io.Externaizable` needs to invoke the no-argument constructor to de-serialize the instances.

Both, `java.io.Serializable` and `java.io.Externaizable` share the same limitation. For each of the de-serialized instance, there is a separate Java object created. This means that if a large number of instances are de-serialized (which is typical during the "Reduce" phase) a large number of objects are created. This puts pressure on the Garbage Collection processes. However, most of the time only the current instance is needed for processing when de-serializing from a stream of instances. The GC pressures can be relived significantly by reusing instances to de-serialize the byte stream.

The [Kryo](https://github.com/EsotericSoftware/kryo) library evolved to solve these problems.  While Spark supports the default Java Serialization, it is recommended you use the Kryo scheme for performance. Kryo improves on the above two methods as follows :

1. **Firstly, Kryo provides an option to store an integer index representing the classes** being serialized. Since a fully qualified class name includes the package it belongs to, this significantly drops the storage overhead when serializing a class. The drawback is, you have given up portability since you must refer to the class with the same integer index when de-serializing the instances. In practice this is rarely an issue.
2. **Secondly, Kryo supports pooling of instances** during de-serialization process. Which means, a very small number of instances will occupy the memory even if millions of instances are being de-serialized at a time.


However, the schema is still stored in each instance when using Kryo. If a stream only contains instances of **concrete** class, this can be improved further. The class meta-data needs to be stored only once at the start of the stream followed by a byte-stream of serialized attributes for all instances in the stream. During the de-serialization process only a single instance needs to be created and the byte-stream needs to be read into it one instance at a time. After working with the currently de-serialized instance, the next instance byte-stream can be read into the same instance created at the start of the de-serialization process. The Hadoop `org.apache.hadoop.io.Writable` follows this approach.  Since this is common Big-Data processing pattern, this approach has been adopted in Spark (DataFrames have the same schema for all rows) and Flink.

It needs to be emphasized that when using the last discussed scheme, you must work only with concrete classes. [Polymorphism and Big-Data do not mix well](http://www.bigsynapse.com/polymorphism-and-mapreduce). If the class of the instance can only be determined at run-time, the class meta-data needs to be included in the serialized byte-stream.  The above explanation should make it clear why `java.io.Serializable` or `java.io.Externalizable` use bloated schemes. Both schemes support polymorphic behavior. And this is the main reason why the `setMapperOutputKeyClass`, `setMapperOutputValueClass`, `setOutputKeyClass` and `setOutputValueClass` methods need to be explicitly invoked on the Job instance when using Hadoop MapReduce. Those calls are annoying from a programmer perspective but are necessary to achieve high performance. These invocations are what allows the instances to be serialized without their schema definitions.

Figure 2 shows how various schemes contribute to storage overheard and impact GC performance.

![Silvrback blog image](https://silvrback.s3.amazonaws.com/uploads/f11eefc8-c78b-4914-b797-0a0290f04775/SerializationDifferences_large.png)

*Figure 2: Evolution of Java Serialization Schemes* 

## Engineering High Performance - How Frameworks do it? ##

The following techniques enable a high-performance distributed platform-

1. **Framework specific serialization scheme** to reduce the memory/storage footprint of serialized instances. This also leads to a more efficient use of the Memory Hierarchy. New frameworks (Spark, Flink) actively utilize the [sun.misc.Unsafe](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/) package for this purpose.

2. **Take control over memory management** - The most common technique is to allocate/release memory in large chunks. Firstly, this avoids heap-fragmentation leading to a more efficient and complete use of Memory. Secondly, it reduces the amount of objects in the Tenured Generation which in turn allows Major GC to complete faster. HBase uses this technique. [Flink takes it to a new level altogether](https://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html) by trying to avoid major GC altogether.  

3. **Achieve more efficient utilization of the memory hierarchy** by maximizing the use of  L1/L2/L3 caches. This technique refers to using only the data needed to perform the operation. For example, the sort operation only needs the sort attributes. If entire object is loaded in memory it is more likely to lead to frequent cache evictions. This causes the CPU to become the bottleneck since the CPU loses more cycles waiting for data to be read back from main-memory than from the L1/L2/L3 caches. If we can only load the attributes of the instances needed for a specific operation ("Sort" in this case) it would improve CPU utilization. A common operation at Big Data scale is "Comparing two instances" (required for the "Sorting" phase which is the key operation for "Join"/"Reduce"). If comparison can be applied on Binary Data there is no need to serialize the byte stream into a separate instance. All Big Data frameworks apply this. The most common Hadoop Data type [`org.apache.hadoop.io.Text`](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/Text.html)  utilizes this technique via its super-class [`org.apache.hadoop.io.BinaryComparable`](https://hadoop.apache.org/docs/current/api/src-html/org/apache/hadoop/io/BinaryComparable.html#line.57).

4. **Utilize Off-Heap memory** - This option utilizes memory outside the Java Heap using [Java NIO](http://tutorials.jenkov.com/java-nio/index.html) or custom implementation based on the [sun.misc.Unsafe](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/) package. The goal is to provide interfaces to access the memory not managed by the Java Heap. This memory is never Garbage Collected and hence allows GC to run extremely fast. On the downside, the programmer has to assume responsibility for managing the memory allocated via this route. HBase provides an Off-Heap memory option for its Block Cache. Spark utilizes Off-Heap option during [Shuffle and Cache block transfer process](http://spark.apache.org/docs/latest/configuration.html). Flink is [currently working](https://github.com/apache/flink/pull/290) on providing an Off-Heap option for its Management Memory sub-system.


### Framework specific Serialization Schemes ###

We discussed this briefly earlier. Let us delve into more details in this section. 

The reason specialized Serialization techniques work at Big Data scale is because streams of processed data typically have the same data-type. We can achieve the following optimization's using a custom serialization schemes-

1. Store the exact schema (no [polymorphism](http://www.bigsynapse.com/polymorphism-and-mapreduce)) only once per dataset 
2. Store byte streams per tuple, with offsets to instance attributes (See following [presentation](http://www.slideshare.net/SparkSummit/deep-dive-into-project-tungsten-josh-rosen) for a Spark example) 

Typically when you need to read an instance attribute we need to de-serialize the entire instance and then make `getXXX` call on the instance to retrieve the desired attribute. Custom Serialization scheme's can allow us to extract just the bytes for the required instance attribute and de-serialize the only the required attribute. If an instance has a large number of attributes, this method allows us to extract only the attributes we need. It reduces the number of instances generated by orders of magnitude which in turn will contribute to an efficient utilization of the memory hierarchy (L1/L2/L3 access over a main-memory fetch). Since "Sort", "Shuffle" and "Join" operations require only a small number of attributes, these process can be made very efficient with respect to memory and CPU access.

#### Hadoop Solution - org.apache.hadoop.io.Writable####

This topic was covered earlier. Refer to Figure 2 for more details.

#### Spark Solution - Project Tungsten####

Spark encouraged the use of Kryo while supporting Java Serialization. Both have the advantage of supporting the full blown Object Oriented Model for Spark data types. The size of serialized types is considerably higher (Kryo supports a more efficient mechanism since the data types can be encapsulated in an integer. However every instance needs to carry this integer with it which is 4 byte overhead.).

Project Tungsten finally acknowledged that while the above approach is pure, the purity must be sacrificed for performance gains. The following [presentation](http://www.slideshare.net/SparkSummit/deep-dive-into-project-tungsten-josh-rosen) provides an overview of Project Tungsten approach along with an example.  

The key idea exploits the following characteristics of a typical dataset in Spark-

1. Datasets are usually tuples of with identical schema (if you imagine a dataset as an RDBMS based Resultset, this becomes obvious)
2. Storing Type information for each Tuple is wasteful. Storing data types for each component of the Tuple is even more wasteful. It is cheaper and more efficient to store the schema once and use it many times

The key idea is captured in the slide which says-
> Generality has a cost, so we should use semantics and schema to exploit specificity instead

Tungsten will also apply custom serialization to improving the performance of Shuffle. The Shuffle process is impacted more by the efficiency of the serialization scheme than the time taken to send it over the network.

Compact custom serialization schemes will save the time it takes to perform the shuffle by making the serialization process faster as well as reducing the byte-loads needed to transfer over the network. Read the Code Generation section in the following [blog entry](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html) for more details. The blog shows that custom serialization based shuffle is twice as fast as Kryo.

> If you want to squeeze the last bit of performance from your application you will need to exploit the characteristics of your application (how the data arrives or how the data is structured). 
> 

#### Flink Solution ####

Flink it designed ground up to support a highly efficient serialization scheme. Firstly, like Spark, Flink supports arbitrary Java and Scala types. This is a big improvement over data types having to implement a specific interface like in Hadoop (`org.apache.hadoop.io.Writable`). Flink utilizes Reflection to identify the return types of user defined functions. Scala programs are analyzed using the Scala compiler. Flink utilizes a custom class called [TypeInformation](https://github.com/apache/flink/blob/release-0.9.0-milestone-1/flink-core/src/main/java/org/apache/flink/api/common/typeinfo/TypeInformation.java) which represents *any* data type. TypeInformation supports the following types-

- **BasicTypeInfo**: Any (boxed) Java primitive type or java.lang.String.
- **BasicArrayTypeInfo**: Any array of a (boxed) Java primitive type or java.lang.String.
- **WritableTypeInfo**: Any implementation of Hadoop’s Writable interface.
- **TupleTypeInfo**: Any Flink tuple (Tuple1 to Tuple25). Flink tuples are Java representations for fixed-length tuples with typed fields.
- **CaseClassTypeInfo**: Any Scala CaseClass (including Scala tuples).
- **PojoTypeInfo**: Any POJO (Java or Scala), i.e., an object with all fields either being public or accessible through getters and setter that follow the common naming conventions.
- **GenericTypeInfo**: Any data type that cannot be identified as another type.
(Above snippet has been taken from the following blog entry -[ Juggling with Bits and Bytes](https://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html)).

Each of the above `TypeInformation` implementations provides a custom serializer. The `GenericTypeInfo` is the `TypeInformation` of last resort and delegates to Kryo. For more details read the section - How does Flink serialize objects? at the fore-mentioned [blog](https://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html) entry. 

`TypeInformation` also provides custom `TypeComparators` which provide more efficient ways to compare data types in binary format. This allows efficient comparison and sorting for data types which can be used as keys. `TypeInformation` is a public API in Flink and can be extended to support custom `TypesInformation`, Serializer's and Comparator's for more efficient handling of user defined types.

`TypeInformation` also integrates with the Flink's custom Memory Management (backed by `MemorySegment` which we will look at soon). This utilizes the highly efficient Java Unsafe operations. 

> Flink neatly combines a highly efficient serialization mechanism which is extensible, defaults gracefully (to Kryo) when it lacks information, is extensible to custom data types and integrates with its highly efficient Memory Management module designed to support avoidance of major Garbage Collection leading to extremely high throughputs.
>    


### Take Control over Memory Management###

Major Garbage Collection is one of the biggest constraints on achieving high performance. This constraint conflicts with a common technique used to achieve high performance - caching of datasets in memory. Too much of cache can trigger major collection often leading to poor throughput. In the worst case it can cause an OutOfMemoryError.  Excessive use of memory also leads to more fragmentation of memory leading to the dreaded OutOfMemoryError even when the total available memory exceeds the programs memory requirements due to not enough contiguous memory being free.

#### Memory Management in HBase ####

Taking control over Memory Management is not new to Spark or Flink. HBase has been doing it as well. This section will provide a perspective on early Memory Management systems in Big Data applications. It will provide the foundation for how Flink and Spark (Tungsten) are adopting custom Memory Management techniques to achieve high performance.

HBase uses Memstore as its in-memory data store and periodically flushes to disk. In the Memstore HBase allocates memory in chunks of 2MB. This technique is described in detail in this 3 part blog by Cloudera- Avoiding Full GCs in Apache HBase with MemStore-Local Allocation Buffers

Part 1 - [http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/](http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/)

Part 2 - [http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-2/](http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-2/)

Part 3 - [http://blog.cloudera.com/blog/2011/03/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-3//](http://blog.cloudera.com/blog/2011/03/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-3//)

The articles go into considerable detail on Java Garbage Collection and how HBase is engineered to work within the constraints imposed by the Java Garbage Collection system. 

The HBase approach can be summarized as follows:

1. HBase is a key-value store which stores key-value pairs as byte arrays 
2. Most key-value pairs are small. The goal is to avoid creating a separate byte arrays for each key-value pairs
3. When a key-value pair is first stored in HBase, it is placed in an in-memory store called Memstore which has budget on how much memory it can utilize. When the budget thresholds are exceeded the Memstore is flushed to disk. Read this [excellent blog entry](http://blog.sematext.com/2012/07/16/hbase-memstore-what-you-should-know/) from Sematext for more details. 
4. Within the thresholds, Memstore allocates memory in 2MB chunks into which the in-memory key-value pairs are stored initially. When a 2MB chunk is exhausted a new one is allocated and used. 

This technique utilizes memory efficiently. There are less holes created as memory is allocated in chunks of 2MB each. GC sweeps are faster as it is faster to determine if 2MB chunk is eligible for collection vs. several smaller chunks representing key-value instances. Also memory is reclaimed in larger chunks 2MB each making it easier to re-allocate by the JVM when needed. A 2000 separate 1 KB segments in the memory-map are harder to allocate than a 2MB segment.


#### Memory Management in Flink ####

Flink does something similar to HBase except on a much larger scale. Flink divides the Java heap memory into three regions-

- **Network Buffers** - By default 2048, 32 KB buffers are allocated at startup to buffer records at startup. The number of buffers can be adjusted using the parameter `taskmanager.network.numberOfBuffers`
- **Memory Manager Pool** - This is a large collection of 32KB buffers managed by the [MemoryManager](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/runtime/memorymanager/MemoryManager.java) instance. Each buffer is an instance of [MemorySegment](https://github.com/apache/flink/blob/release-0.9.0-milestone-1/flink-core/src/main/java/org/apache/flink/core/memory/MemorySegment.java). The MemoryManager pools MemorySegments to algorithms (such as Sort/Shuffle/Join) who allocate and release MemorySegment instances using the single instance of MemoryManager. Between the Memory Manager Pool and the Network Buffers approximately 70% of the heap memory is shared
- **Memory for user code** - The remaining memory is left for the user code. From a GC perspective it represents the Young Generation regions. It is meant to be used by short-lived instances produced by the user code.


The benefits of the above approach are-

1. Short lived instances live in the last segment and are collected quickly.

2. Algorithms request memory in segments of 32KB and release it in segments of 32KB. However the memory is never released from a GC perspective. `MemoryManager` instance still holds a reference to the `MemorySegment` instance allocated or released by the application. 

3. Algorithms have a strict budget of how many Memory Segments they can use. If an Algorithm needs more than its budget, it will need to persist unused segments to disk and recover them back from disk when it needs them back. If the algorithm needs more memory than its allowed budget, it persists some segments to disk and reuses the MemorySegment. 

Once again we see how "Generality has to be sacrificed for Specificity to achieve high performance". To achieve better throughput and performance we had to adopt more complexity in our code. Figure 3 demonstrates the Flink approach to Memory Management

![Silvrback blog image](https://silvrback.s3.amazonaws.com/uploads/4e56ce43-b6d3-4a3e-a8e0-da4be8a3b735/FlinkMemoryManagement_large.png)

*Figure 3: Flink Memory Management*


Algorithms request/release memory in segments of 32KB (default size for `MemorySegment` storage). However the memory is never released from a GC perspective. Most user-defined objects are short-lived and collected during the minor collection phase. As long as the program does not attempt to create custom cache's of its own, the amount of memory in the Tenured Generation remains constant. Consequently the Major collection is never triggered. This vastly improves the throughput of the overall Flink system. **This is the key point** - A programmer should not attempt to create simplistic caching schemes for long-lived instances. This will fill up the tenured generation and trigger a major GC or OutOfMemoryError in the worst case. If the programmer needs to create such caches she needs to assume responsibility for managing memory using the `MemoryManager`. 

For more details refer to the following two articles 

1. [Memory Management (Batch API) in Flink](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=53741525)
2. [Juggling with Bits and Bytes](https://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html) 

### Achieve more efficient utilization of the Memory Hierarchy ###

With networks getting faster, the CPU is fast becoming the bottleneck at Big Data scale. Reading data off the L1/L2/L3 is orders of magnitude faster than reading data off main memory. Profiling applications lead to the conclusion that a large fraction of CPU time is spent waiting for data to arrive from the main memory. If the same data could arrive from L1/L2/L3 caches this wait would be substantially reduced and the algorithms would experience a substantial increase in throughput.

As discussed earlier custom serialization supports working directly with binary data to avoid having to de-serialize entire instances. *Custom Serialization schemes provides fine-grained control over how the bytes of an instance are laid out in memory. This allows referencing of specific attributes of an instance directly without having to de-serialize the entire instance into memory.* For operations such as "Sorting" it allows algorithms to work with only the sort-key+pointer to the instance, substantially improving the efficiency of the CPU. 

#### Flink Solution ####

Consider how Flink performs sorting of instances. This is an extremely common operation. Assume that Flink needs to sort a dataset by a single integer attribute of the dataset. Flink achieve high performance as follows:

1. The dataset is stored in two seperate memory segments	
	- The bytes representing the full data are stored in one set of memory segments. Most datasets have the same schema. Hence Flink optimizes the storage as discussed in the section on "Framework Specific Serialization Schemes"
	- A separate set of segments is used to store two attributes per element of the dataset - The pointer to the dataset element followed by the sort key (integer in our case). 

2. Sorting can now be performed on the set comprising the pointer and the sort attribute. It is far more likely that the L1/L2/L3 cache usages will be maximized in this approach as the amount of data needed to sort completely is very limited.

The other advantages of the above approach are-

1. Flink can operate directly on binary data. There is no need to de-serialize the sort attribute if it is possible to operate directly on the byte arrays. [Remember](https://hadoop.apache.org/docs/current/api/src-html/org/apache/hadoop/io/BinaryComparable.html#line.57) how the `org.apache.hadoop.ioText` compares two instances using only the byte arrays.

2. There is no need to de-serialize the data maintained in the Memory Segment which stores the *entire* elements of the dataset. This dataset can be safely persisted to disk until it is needed.

For more details on how Flink works with bits and bytes directly refer to the following excellent blog articles-

1. [Juggling with Bits and Bytes](https://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html) 
2. [Peeking into Apache Flink's Engine Room](https://flink.apache.org/news/2015/03/13/peeking-into-Apache-Flinks-Engine-Room.html) 

#### Spark Solution - Project Tungsten ####

Spark is similar to Flink with respect to sorting algorithms. More details are available by following the blog entry - [Project Tungsten: Bringing Spark Closer to Bare Metal](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html) 

### Utilize Off-Heap memory ###

Utilizing Off-Heap memory is the last resort to squeeze out maximum performance in Java. Java provides libraries which allow programmers to explicitly manage memory. Managed heap is the feature of the Java programming language which relieves the programmer of the complex and error prone task of performing memory management. However, it also has drawbacks. Firstly, as a programmer you have no control over how frequently the Garbage Collection runs and how long it runs. Also you can run into the "OutOfMemoryError" if you indiscrimately cache instances. As discussed earlier, the managed heap runs out of space even though there may be plenty of memory available on the machine due to memory fragmentation. 

You are not forced to use only the managed memory in Java. By utilizing the `java.nio`  or the `sun.misc.misc.Unsafe` packages, the programmer has access to the unmanaged memory. The drawback is, the programmer is responsible for allocating and releasing memory. The advantage is, the memory managed through this process does not come under the purview of the Garbage Collection processes which significantly reduces the load on the Garbage Collection which returns quickly. The trade off is - you adopt higher development complexity to achieve predictable/high throughput. 
 
Although not commonly used when building custom applications, high performance computing frameworks have been utilizing Off-Heap memory for long time. Terracotta utilizes unmanaged memory in its [BigMemory](http://www.terracotta.org/products/bigmemory) product. BigMemory allows you caches to grow to GB levels. 

[OpenHFT](https://github.com/OpenHFT) provides the [Chronicle library](https://github.com/OpenHFT/Chronicle-Engine) which supports high performance, low latency, reactive processing framework. This framework makes extensive use of Off-Heap memory.

HBase supports the Off-Heap option for its [Block Cache](http://hortonworks.com/blog/hbase-blockcache-101/).

Flink is [developing](https://github.com/apache/flink/pull/290) a MemoryManager which will utilize the OffHeap Memory. See this [entry](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=53741525) for more details.

Spark supports Off-heap storage (experimental) for RDD caching through Tachyon.

I have developed a library called [LargeCollections](https://github.com/sameeraxiomine/LargeCollections) which utilizes [LevelDB](https://github.com/google/leveldb) to support a massive off-heap Hashmap. This project uses a JNI wrapper around C++ implementation of LevelDB.

Ultimately the goal of all frameworks is to achieve the MPP levels of performance. Off-heap usage brings the Java Programmer one step closer to using Java like the C programming language. 

## Applying these lessons to your own Big Data Applications ##

Even though the engineering methods described above might seem exotic the lessons can be applied to your own Big Data applications. Some of the techniques I have applied are-

1. Develop your own compact Serialization schemes. This reduces the I/O between tasks.
  
2. Work with binary data as much as possible. Avoid going from binary to object and back to binary as much as possible. CPU is indeed the bottleneck at large data scales. Every time you work with `org.apache.hadoop.io.Text` you are already doing this. Look for opportunities to employ the technique utilized by `org.apache.hadoop.io.Text` for comparison to your own key-value classes. 

3. Work with minimum possible data at all times. If only a fraction of your data contributes to the bulk of your processing work with only that much data for the most part and do an expensive join at the end. This way you avoid having to send unnecessary amounts of data over the network or de-serializing too many large object back into memory.

4. Do not shy from using Off-heap memory. Sometimes using it can aid high performance techniques like Map-Side join. If one side of your join datasets is of moderate quantity, storing that side in Off-Heap memory such that it can be easily looked up by the join key allows the join to work entirely on the Mapper node without the expensive Reduce operation (which requires shuffle and sort). This is a variation of the broadcast joins discussed in this article. I have developed the [LargeCollections](https://github.com/sameeraxiomine/LargeCollections) to provide a simple interface to work with Off-heap memory. It takes away the complexity of working with the Usafe or the NIO libraries and provides the programmer with the simple Hashmap abstraction they are familiar with. It supports efficient data types like org.apache.hadoop.io.Writable types. It also supports development of custom Serializer/De-Serializer (SerDe) to support custom serialization schemes. Since it uses memory which is off the java heap, it substantially reduces the pressure on the Java Memory Management processes.  

## Acknowledgements ##

My goal in this paper was to aggregate the fascinating evolution of performance engineering techniques utilized by Big Data Frameworks to achieve C-levels of performance. If you think I missed something or made and error send me a note at sameer@axiomine.com and I will be happy to include/correct the information.

I would like to thank [Slim Baltagi](https://www.linkedin.com/profile/view?id=21979395), [Fabian Hueske](https://www.linkedin.com/profile/view?id=4767856) and [Stephan Ewen](https://www.linkedin.com/profile/view?id=4327904) for taking the time to review the draft version of this article and offering critical feedback.

## Links ##
[[1](http://0x0fff.com/spark-dataframes-are-faster-arent-they/ "spark-dataframes-are-faster-arent-they")] - Frameworks are getting faster but not as fast as MPP (Spark Dataframes) 

[[2](https://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html "Juggling-with-Bits-and-Bytes.html")] - Flink - Juggling with Bits and Bytes
 
[[3](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=53741525 "Memory Management API in Flink")] - Flink Internals - Memory Management API


[[4](https://flink.apache.org/news/2015/03/13/peeking-into-Apache-Flinks-Engine-Room.html "peeking-into-Apache-Flinks-Engine-Room")] - Apache Flink - How custom Memory Management helps achieve high performance using Joins.

[[5](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html "project-tungsten-bringing-spark-closer-to-bare-metal")] - Project Tungsten: Bringing Spark Closer to Bare Metal

[[6](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html "deep-dive-into-spark-sqls-catalyst-optimizer")] - Deep Dive into Spark SQL’s Catalyst Optimizer (Code Generation and Performance)

[[7](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/ "Usafe Java")] - sun.misc.misc.Unsafe usage for C style memory management

[[8](http://howtodoinjava.com/2013/10/19/usage-of-class-sun-misc-unsafe/ "More Usafe Java")] - sun.misc.misc.Unsafe usage for C style memory management - How to do it.

[[9](http://en.wikipedia.org/wiki/Dynamic_dispatch "Dynamic Dispatch for Polymorphic Operations")] - Dynamic Dispatch for Polymorphic Operations

[[10](http://shipilev.net/blog/2015/black-magic-method-dispatch/ "Black Magic of Java Method Dispatch")] - Black Magic of Java Method Dispatch

[[11](http://www.slideshare.net/SparkSummit/deep-dive-into-project-tungsten-josh-rosen "deep-dive-into-project-tungsten-josh-rosen")] - Deep Dive into Project Tungsten

[[12](http://www.javamex.com/tutorials/memory/object_memory_usage.shtml "object_memory_usage")] - Object Memory Usage

[[13](http://www.slideshare.net/cnbailey/memory-efficient-java "Java Object Size")] - True Size of a Java Object

[[14](http://events.linuxfoundation.org/sites/events/files/slides/flink_apachecon_small.pdf "Flink Big Data Analytics Platform")] - Flink Big Data Analytics Platform

[[15](http://www.slideshare.net/KostasTzoumas/flink-internals "Flink Internals")] - Flink Internals

[[16](http://www.confluent.io/blog/real-time-stream-processing-the-next-step-for-apache-flink/ "Real time stream processing in Flink")] - Real time stream processing in Flink

[[17](http://www.slideshare.net/ydn/flink-yahoo-meetup "Flink - Fast and reliable large-scale data processing")] - January 2015 HUG: Apache Flink: Fast and reliable large-scale data processing

[[18](http://hortonworks.com/blog/hbase-blockcache-101/ "HBase Block Cache")] - HBase BlockCache 101

[[19](http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/ "HBase Performance Engineering Part 1")] - Avoiding full gcs in hbase with memstore local allocation buffers part 1

[[20](http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-2 "HBase Performance Engineering Part 2")] - Avoiding full gcs in hbase with memstore local allocation buffers part 2

[[21](http://blog.cloudera.com/blog/2011/03/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-3 "HBase Performance Engineering Part 3")] - Avoiding full gcs in hbase with memstore local allocation buffers part 3

[[22](http://blog.sematext.com/2012/07/16/hbase-memstore-what-you-should-know/ "What you need to know about HBase Memstore")] - Avoiding full gcs in hbase with memstore local allocation buffers part 3

[[23](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html "Garbage Collection Basics")] - Basics of Garbage Collection
