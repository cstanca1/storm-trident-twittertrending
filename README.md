#Twitter Trending Topics with Apache Storm 0.10.0 using Trident

## What it does
This project demonstrates how you can build a topology using a custom spout, function, and several built-in functions provided by Trident. It calculates trending topics (hashtags) on Twitter.

This example is based on the [trident-storm (https://github.com/jalonsoramos/trident-storm) example by Juan Alonso.
Decided to run it locally because it requires my Twitter credentials.

##Trident
Trident is a high-level abstraction that is provided by Storm. It supports stateful processing. The primary advantage of Trident is that it can guarantee that every message that enters the topology is processed only once. This is difficult to achieve in a raw Java topology, which guarantee's that messages will be processed at least once. There are also other differences, such as built-in components that can be used instead of creating bolts. In fact, bolts are completely replaced by less-generic components, such as filters, projections, and functions.

The Trident code that implements the topology:
```
topology.newStream("spout", spout)
	.each(new Fields("tweet"), new HashtagExtractor(), new Fields("hashtag"))
	.groupBy(new Fields("hashtag"))
	.persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))
	.newValuesStream()
	.applyAssembly(new FirstN(10, "count"))
	.each(new Fields("hashtag", "count"), new Debug());
```
This does the following:

* Create a new stream from the spout. The spout retrieves tweets from Twitter.

* **HashtagExtractor**, a custom function, is used to extract hashtags from each tweet. These are emitted to the stream.

* The stream is then grouped by hashtag, and passed to an aggregator. This aggregator creates a count of how many times each hashtag has occurred, which is persisted in memory. Finally, a new stream is emitted that contains the hashtag and the count.

* Since we are only interested in the most popular hashtags for a given batch of tweets, the FirstN assembly is applied to return only the top 10 values based on the count field.

Other than the spout and **HashtagExtractor**, we are using built-in Trident functionality.

For information on built-in operations, see <a href="https://storm.apache.org/apidocs/storm/trident/operation/builtin/package-summary.html" target="_blank">storm.trident.operation.builtin</a>.

For Trident-state implementations other than **MemoryMapState**, see the following:

* <a href="https://github.com/fhussonnois/storm-trident-elasticsearch" target="_blank">https://github.com/fhussonnois/storm-trident-elasticsearch</a>

* <a href="https://github.com/kstyrc/trident-redis" target="_blank">https://github.com/kstyrc/trident-redis</a>

###The spout

The spout, **TwitterSpout** uses <a href="http://twitter4j.org/en/" target="_blank">Twitter4j</a> to retrieve tweets from Twitter. A filter is created (love, music, and coffee,) and incoming tweets (status) that match the filter are stored into a <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/LinkedBlockingQueue.html" target="_blank">LinkedBlockingQueue</a>. Finally, items are pulled off the queue and emitted into the topology.

###The HashtagExtractor

To extract hashtags, <a href="http://twitter4j.org/javadoc/twitter4j/EntitySupport.html#getHashtagEntities--" target="_blank">getHashtagEntities</a> is used to retrieve all hashtags contained in the tweet. These are then emitted to the stream.


##Get a twitter account

From https://dev.twitter.com/app, get: 
```
	oauth.consumerSecret
	oauth.consumerKey
	oauth.accessToken
	oauth.accessTokenSecret
```
Set them in resources/twitter4j.properties file. Without these, the topology will not work.

##Build the topology

Use the following build the project.
```
		cd TwitterTrending
		c:\maven\bin\mvn compile
```
##Test the topology

Use the following command to test the topology locally.
```
	cd TwitterTrending
	c:\maven\bin\mvn compile exec:java -Dstorm.topology=com.cristi.storm.TwitterTrendingTopology
```
After the topology starts, you should see debug information containing the hashtags and counts emitted by the topology. The output should appear similar to the following.
```
	DEBUG: [Quicktellervalentine, 7]
	DEBUG: [GRAMMYs, 7]
	DEBUG: [AskSam, 7]
	DEBUG: [poppunk, 1]
	DEBUG: [rock, 1]
	DEBUG: [punkrock, 1]
	DEBUG: [band, 1]
	DEBUG: [punk, 1]
	DEBUG: [indonesiapunkrock, 1]
```
In the Storm CLI, a Storm or Trident topology can be submitted with a command similar to:

```
$ bin/storm jar {path-to-uber-jar} {topology-submission-main-class} {arg1} {arg2} ...
```
In our specific case, the same can be submitted after building the jar with maven with all dependencies included
```
cd C:\eclipse_workspace\TwitterTrending\target
C:\Storm\apache-storm-0.10.0\bin\storm jar TwitterTrending-1.0-SNAPSHOT-jar-with-dependencies.jar com.cristi.storm.TwitterTrendingTopology
```

On the cluster, it can be submitted similarly, but since Storm is already there, storm depedency in the packaged jar should be excluded and storm classpath from the server should be added.

Example submit topology to HDP 2.3 cluster:
```
sudo -u storm -s
/usr/hdp/2.3.4.0-3485/storm/bin/storm jar /home/storm/TwitterTrending-1.0-SNAPSHOT-nostorm.jar com.cristi.storm.TwitterTrendingTopology -c nimbus.host=ip-172-31-56-133.ec2.internal
```

## Useful links:
Writing Data to HDFS with the Storm-HDFS Connector:
http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.0/bk_storm-user-guide/content/writing-data-with-storm-hdfs-connector.html

Both the core-storm and Trident APIs support streaming data directly to Apache Hive using Hive transactions. Data committed in a transaction is immediately available to Hive queries from other Hive clients. Storm developers stream data to existing table partitions or configure the streaming Hive bolt to dynamically create desired table partitions. Currently, data may be streamed only into bucketed tables using the ORC file format. See: http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.0/bk_storm-user-guide/content/ch_storm-streaming-to-hive.html





