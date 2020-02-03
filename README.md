# Elasticsearch

## Architecture

* Distributed schema-less JSON document store
* Each node may play one or more of the following roles
  * Master-eligible node
  * Voting-only node
  * Data node
  * Ingest node
  * ML node
  * Co-ordinating node (cannot be disabled)
* By default each node is eligible to play any of the roles
* However, in large clusters, some nodes play a dedicated role
* High availability (HA) clusters require at least three master-eligible nodes, at least two of which are not voting-only nodes. Such a cluster will be able to elect a master node even if one of the nodes fails
* Role of master node
  * Maintain the global cluster state
  * Reassign shards when nodes join or leave the cluster
* On startup, nodes discover master nodes by connecting to the seed nodes configured in its `elasticsearch.yml`, verifying if they are master eligible and sharing the details of its known master-eligible peers
* If the node itself is not master-eligible then it continues this discovery process until it has discovered an elected master node
* If the node itself is master-eligible then it continues this discovery process until it has either discovered an elected master node or else it has discovered enough masterless master-eligible nodes to complete an election
* If neither of these occur quickly enough then the node will retry after `discovery.find_peers_interval` which defaults to 1s
* Only the master node is allowed to update the metadata representing the state of the whole cluster - cluster state
* Cluster state includes information such as
  * The set of nodes in the cluster
  * All cluster-level settings
  * Information about the indices in the cluster, including their mappings and settings
  * The locations of all the shards in the cluster
* The master node processes one batch of cluster state updates at a time, computing the required changes and publishing the updated cluster state to all the other nodes in the cluster
* Once the master has collected acknowledgements from enough master-eligible nodes, the new cluster state is said to be committed and the master broadcasts another message instructing nodes to apply the now-committed state
* Master node waits for `cluster.publish.timeout` (default 30s) for the cluster state to be committed. If this time is reached before the new cluster state is committed then the cluster state change is rejected and the master considers itself to have failed. It stands down and starts trying to elect a new master
* The master waits for the lagging nodes to catch up for a further time, `cluster.follower_lag.timeout`, which defaults to 90s. If a node has still not successfully applied the cluster state update within this time then it is considered to have failed and is removed from the cluster
* Cluster state updates are typically published as diffs to the previous cluster state, which reduces the time and network bandwidth needed to publish a cluster state update
* If a node is missing the previous cluster state, for example when rejoining a cluster, the master will publish the full cluster state to that node so that it can receive future updates as diffs
* Follower check - The elected master periodically checks each of the nodes in the cluster to ensure that they are still connected and healthy - `cluster.fault_detection.follower_check.interval` (default 1s), `cluster.fault_detection.follower_check.timeout` (default 10s) & `cluster.fault_detection.follower_check.retry_count` (default 3)
* Leader check - Each node in the cluster also periodically checks the health of the elected master - `cluster.fault_detection.leader_check.interval` (default 1s), `cluster.fault_detection.leader_check.timeout` (default 10s) & `cluster.fault_detection.leader_check.retry_count` (default 3)
* Elasticsearch allows these checks to occasionally fail or timeout without taking any action. It considers a node to be faulty only after a number of consecutive checks have failed. You can control fault detection behavior with `cluster.fault_detection.*` settings
* If the elected master detects that a node has disconnected, however, this situation is treated as an immediate failure. The master bypasses the timeout and retry setting values and attempts to remove the node from the cluster
* Similarly, if a node detects that the elected master has disconnected, this situation is treated as an immediate failure. The node bypasses the timeout and retry settings and restarts its discovery phase to try and find or elect a new master
* For the cluster to be fully operational, it must have one active master
* Elasticsearch does not have transactions
* All index and delete operations are written to the translog or transaction log after being processed by the internal Lucene index but before they are acknowledged
* Lucene commit is not invoked after every write operation. It is invoked periodically in the background. Therefore in case of a crash, the lucene changes since the last commit are replayed from the translog
* After every Lucene commit, a new translog is started
* By default, `index.translog.durability` is set to `request` meaning that Elasticsearch will only report success of an index, delete, update, or bulk request to the client after the translog has been successfully fsynced and committed on the primary and on every allocated replica. If `index.translog.durability` is set to `async` then Elasticsearch fsyncs and commits the translog only every `index.translog.sync_interval` which means that any operations that were performed just before a crash may be lost when the node recovers
* Text fields are stored in inverted indices, and numeric and geo fields are stored in BKD trees
* Each index in Elasticsearch is divided into shards and each shard can have multiple replicas (replication group)
* Elasticsearch’s data replication model is based on the primary-backup model
* Each replication group has one primary shard and one or more replica shards
* The primary shard serves as the main entry point for all indexing operations. It is in charge of validating them and making sure they are correct. Once an index operation has been accepted by the primary, the primary is also responsible for replicating the operation to the other copies
* A "shard" is the basic scaling unit for Elasticsearch
* The number of shards is specified at index creation time, and cannot be changed later on
* An Elasticsearch index is made up of one or more shards, which can have zero or more replicas
* Each shard is a Lucene index
* A Lucene index is made up of one or more immutable index segments
* As data is written to a shard, it is periodically published into new immutable Lucene segments on disk, and it is at this time it becomes available for querying. This is referred to as a refresh
* As the number of segments grow, these are periodically consolidated into larger segments. This process is referred to as merging
* As all segments are immutable, this means that the disk space used will typically fluctuate during indexing, as new, merged segments need to be created before the ones they replace can be deleted. Merging can be quite resource intensive, especially with respect to disk I/O
* The shard is the unit at which Elasticsearch distributes data around the cluster. The speed at which Elasticsearch can move shards around when rebalancing data, e.g. following a failure, will depend on the size and number of shards as well as network and disk performance
* Avoid having very large shards as this can negatively affect the cluster's ability to recover from failure. There is no fixed limit on how large shards can be, but a shard size of 50GB is often quoted as a limit that has been seen to work for a variety of use-cases
* As segments are immutable, updating a document requires Elasticsearch to first find the existing document, then mark it as deleted and add the updated version. Deleting a document also requires the document to be found and marked as deleted
* For this reason, deleted documents will continue to tie up disk space and some system resources until they are merged out, which can consume a lot of system resources
* Try to use time-based indices for managing data retention whenever possible. Group data into indices based on the retention period. Time-based indices also make it easy to vary the number of primary shards and replicas over time, as this can be changed for the next index to be generated. This simplifies adapting to changing data volumes and requirements
* For each Elasticsearch index, information about mappings and state is stored in the cluster state. This is kept in memory for fast access. Having a large number of indices and shards in a cluster can therefore result in a large cluster state, especially if mappings are large
* In order to reduce the number of indices and avoid large and sprawling mappings, consider storing data with similar structure in the same index rather than splitting into separate indices based on where the data comes from
* The cluster state is loaded into the heap on every node (including the masters)
* Each shard has data that need to be kept in memory and use heap space. This includes data structures holding information at the shard level, but also at the segment level in order to define where data reside on disk. The size of these data structures is not fixed and will vary depending on the use-case
* Small shards result in small segments, which increases overhead. Aim to keep the average shard size between at least a few GB and a few tens of GB. For use-cases with time-based data, it is common to see shards between 20GB and 40GB in size
* As the overhead per shard depends on the segment count and size, forcing smaller segments to merge into larger ones through a forcemerge operation can reduce overhead and improve query performance. This should ideally be done once no more data is written to the index. Be aware that this is an expensive operation that should ideally be performed during off-peak hours
* A good rule-of-thumb is to ensure you keep the number of shards per node below 20 per GB heap it has configured. A node with a 30GB heap should therefore have a maximum of 600 shards, but the further below this limit you can keep it the better
* In Elasticsearch, each query is executed in a single thread per shard. Multiple shards can however be processed in parallel, as can multiple queries and aggregations against the same shard
* Querying lots of small shards will make the processing per shard faster, but as many more tasks need to be queued up and processed in sequence, it is not necessarily going to be faster than querying a smaller number of larger shards
* If using time-based indices covering a fixed period, adjust the period each index covers based on the retention period and expected data volumes in order to reach the target shard size
* If the data volume is not uniform to warrant a fixed period time-based index, rollover and shrink indexes can be used
* The rollover index API makes it possible to specify the number of documents an index should contain and/or the maximum period documents should be written to it. Once one of these criteria has been exceeded, Elasticsearch can trigger a new index to be created for writing without downtime
* If you have time-based, immutable data where volumes can vary significantly over time, consider using the rollover index API to achieve an optimal target shard size by dynamically varying the time-period each index covers. This gives great flexibility and can help avoid having too large or too small shards when volumes are unpredictable
* If you need to have each index cover a specific time period but still want to be able to spread indexing out across a large number of nodes, consider using the shrink API to reduce the number of primary shards once the index is no longer indexed into. This API can also be used to reduce the number of shards in case you have initially configured too many shards
* Lots of data is time based, e.g. logs, tweets, etc. By creating an index per day (or week, month, …), we can efficiently limit searches to certain time ranges - and expunge old data. Remember, we cannot efficiently delete from an existing index, but deleting an entire index is cheap
* When searches must be limited to a certain user (e.g. "search your messages"), it can be useful to route all the documents for that user to the same shard, to reduce the number of indexes that must be searched

## Analyzers

* By default, Elasticsearch uses the standard analyzer for all text analysis
* Analyzer is: Zero or more character filters => A tokenizer => Zero or token filters
* Tokenizers also record the order or relative positions of each term (used for phrase queries or word proximity queries), and the start and end character offsets of each term in the original text (used for highlighting search snippets)
* Custome Analyzer input parameters:
  * tokenizer - A built-in or customised tokenizer (Required)
  * char_filter - An optional array of built-in or customised character filters
  * filter - An optional array of built-in or customised token filters
  * position_increment_gap - When indexing an array of text values, Elasticsearch inserts a fake "gap" between the last term of one value and the first term of the next value to ensure that a phrase query doesn’t match two terms from different array elements. Defaults to 100. See position_increment_gap for more
* Standard Analyzer
  * The standard analyzer divides text into terms on word boundaries, as defined by the Unicode Text Segmentation algorithm. It removes most punctuation, lowercases terms, and supports removing stop words.
* Simple Analyzer
  * The simple analyzer divides text into terms whenever it encounters a character which is not a letter. It lowercases all terms.
* Whitespace Analyzer
  * The whitespace analyzer divides text into terms whenever it encounters any whitespace character. It does not lowercase terms.
* Stop Analyzer
  * The stop analyzer is like the simple analyzer, but also supports removal of stop words.
* Keyword Analyzer
  * The keyword analyzer is a “noop” analyzer that accepts whatever text it is given and outputs the exact same text as a single term.
* Pattern Analyzer
  * The pattern analyzer uses a regular expression to split the text into terms. It supports lower-casing and stop words.
* Language Analyzers
  * Elasticsearch provides many language-specific analyzers like english or french.
* Fingerprint Analyzer
  * The fingerprint analyzer is a specialist analyzer which creates a fingerprint which can be used for duplicate detection
* While Indexing a document, Elasticsearch determines which index-time analyzer to use by checking the following parameters in order:
  * The analyzer mapping parameter of the field
  ```
  PUT my_index
  {
    "mappings": {
      "properties": {
        "title": {
          "type":     "text",
          "analyzer": "standard"
        }
      }
    }
  }
  ```
  * The default analyzer parameter in the index settings
  ```
  PUT my_index
  {
    "settings": {
      "analysis": {
        "analyzer": {
          "default": {
            "type": "whitespace"
          }
        }
      }
    }
  }
  ```
  * If none of these parameters are specified, the standard analyzer is used
* While searching a document, the analyzer to use to search a particular field is determined by looking for:
  * An analyzer specified in the query itself
  * The search_analyzer mapping parameter
  * The analyzer mapping parameter
  * An analyzer in the index settings called default_search
  * An analyzer in the index settings called default
  * The standard analyzer
* Testing an analyzer
  ```
  POST _analyze
  {
    "analyzer": "whitespace",
    "text":     "The quick brown fox."
  }
  ```
* Testing using _analyze api with a combination of a tokenizer, zero or more character filters and zero or more token filters
  ```
  POST _analyze
  {
    "tokenizer": "standard",
    "filter":  [ "lowercase", "asciifolding" ],
    "text":      "Is this déja vu?" 
  }
  ```
* Normalizers are similar to analyzers except that they may only emit a single token. As a consequence, they do not have a tokenizer and only accept a subset of the available char filters and token filters. Only the filters that work on a per-character basis are allowed
* Character Filter - A character filter receives the original text as a stream of characters and can transform the stream by adding, removing, or changing characters
* Token Filter - Token filters accept a stream of tokens from a tokenizer and can modify tokens (eg lowercasing), delete tokens (eg remove stopwords) or add tokens (eg synonyms)
* Analyzer `standard` doesn't do stemmimg - "laughing", "foxes", "lazy" etc. will remain as is
* Analyzer `standard` doesn't remove stop words unless configured explicitly using parameter `"stopwords": "_english_"`
* Analyzer `english` does stemming and also removes stop words like "the", "an" etc.
* Both `standard` and `english` analyzers have lower case token filter


## Key Parameters

* `cluster.no_master_block` - Specifies which operations are rejected when there is no active master in a cluster. Possible values are `all` and `write`
* `cluster.routing.allocation.enable` - Enable or disable allocation for specific kinds of shards in the node - `all`, `primaries`, `new_primaries` and `none`
 

## Write Model

* **Happy Path**
  * Every indexing operation in Elasticsearch is first resolved to a replication group using routing typically based on the document ID
  * Once the replication group has been determined, the operation is forwarded internally to the current primary shard of the group
  * The primary shard is responsible for validating the operation, execute the operation locally and forwarding it to the other replicas
  * Since replicas can be offline, the primary is not required to replicate to all replicas. Instead, Elasticsearch maintains a list of shard copies thatshould receive the operation. This list is called the in-sync copies and is maintained by the master node
  * As the name implies, these are the set of "good" shard copies that are guaranteed to have processed all of the index and delete operations that have been acknowledged to the user
  * The primary is responsible for maintaining this invariant and thus has to replicate all operations to each copy in this set
  * Once all replicas have successfully performed the operation and responded to the primary, the primary acknowledges the successful completion of the request to the client
* **Failures**
  * If all other shards fail, the primary shard informs the master. Thus the master will not promote and ou-of-date shard to be a primary shard
  * The master monitors the health of the nodes and if it finds a node has died, it will demote the primary and promote some other replica as primary
  * If the primary cannot replicate an operation in a replica, it will inform the master to remove the replica from the In-sync list. Once the primary receives a response from the master about the removal of the failed replica from the In-sync list, it acknowledges the write to the client
  * The master will also instruct another node to start building a new shard copy in order to restore the system to a healthy state
  * Operations that come from a stale primary will be rejected by the replicas
  * When the primary receives a response from the replica rejecting its request because it is no longer the primary then it will reach out to the master and will learn that it has been replaced. The operation is then routed to the new primary

## Read Model

* When a read request is received by a node, that node is responsible for forwarding it to the nodes that hold the relevant shards, collating the responses, and responding to the client. We call that node the coordinating node for that request
* Resolve the read requests to the relevant shards. Note that since most searches will be sent to one or more indices, they typically need to read from multiple shards, each representing a different subset of the data
* Select an active copy of each relevant shard, from the shard replication group. This can be either the primary or a replica. By default, Elasticsearch will simply round robin between the shard copies
* Send shard level read requests to the selected copies
* Combine the results and respond. Note that in the case of get by ID look up, only one shard is relevant and this step can be skipped


* By default, Elasticsearch indexes all data in every field
* Each indexed field has a dedicated, optimized data structure
* When dynamic mapping is enabled, Elasticsearch automatically detects and adds new fields to the index mapping them to the appropriate Elasticsearch datatypes
* It’s often useful to index the same field in different ways for different purposes. For example, you might want to index a string field as both a text field for full-text search and as a keyword field for sorting or aggregating your data. Or, you might choose to use more than one language analyzer to process the contents of a string field that contains user input
* The analysis chain that is applied to a full-text field during indexing is also used at search time. When you query a full-text field, the query text undergoes the same analysis before the terms are looked up in the index
* Elasticsearch aggregations enable you to build complex summaries of your data and gain insight into key metrics, patterns, and trends
* Aggregations are calculated based on the users' search criteria
* You can add servers (nodes) to a cluster to increase capacity and Elasticsearch automatically distributes your data and query load across all of the available nodes
* The more shards, the more overhead there is simply in maintaining those indices
* The larger the shard size, the longer it takes to move shards around when Elasticsearch needs to rebalance a cluster
* Querying lots of small shards makes the processing per shard faster, but more queries means more overhead, so querying a smaller number of larger shards might be faster
* **General Recommendation**
  * Aim to keep the average shard size between a few GB and a few tens of GB. For use cases with time-based data, it is common to see shards in the 20GB to 40GB range
  * Avoid the gazillion shards problem. The number of shards a node can hold is proportional to the available heap space. As a general rule, the number of shards per GB of heap space should be less than 20
* For performance reasons, the nodes within a cluster need to be on the same network. Balancing shards in a cluster across nodes in different data centers simply takes too long
* Cross-cluster replication (CCR) provides a way to automatically synchronize indices from your primary cluster to a secondary remote cluster that can serve as a hot backup
* Cross-cluster replication is active-passive. The index on the primary cluster is the active leader index and handles all write requests. Indices replicated to secondary clusters are read-only followers
* Start elasticsearch
```
export ES_PATH_CONF=<location of config for node1>; ./elasticsearch
export ES_PATH_CONF=<location of config for node2>; ./elasticsearch
export ES_PATH_CONF=<location of config for node3>; ./elasticsearch
```
* Replica shards must be available for the cluster status to be green. If the cluster status is red, some data is unavailable

## Operations

* The number of shards **cannot** be changed once the index is created
* The mappings **cannot** be changed once the index is created
* The number of replicas **can** be changed even after the index is created. The same request can be used to change other index settings
  ```
  PUT /<index_name>/_settings
  {
    "index" : {
        "number_of_replicas" : 2
    }
  }
  ```
*

## Sample API Calls

### Creating an Index

```
PUT /mytestindex
{
  "settings": {
    "analysis": {
      "analyzer": {
        "phrase_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "filter_shingle",
            "lowercase",
            "english_stemmer"
          ]
        }
      },
      "filter": {
        "filter_shingle": {
          "type": "shingle",
          "max_shingle_size": 3,
          "min_shingle_size": 2
        },
        "english_stemmer" : {
          "type" : "stemmer",
          "name" : "english"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "notes": {
        "type": "text",
        "analyzer": "english",
        "fields": {
          "phrase": {
            "type": "text",
            "analyzer": "phrase_analyzer"
          }
        }
      },
      "name": {
        "type": "text"
      },
      "name_completion": {
        "type": "completion"
      },
      "billgroup": {
        "type": "integer"
      }
    }
  }
}
```

```
{
  "suggest": {
    "text": "I hear you gvei yourreason",
    "my-suggest-1": {
      "phrase": {
        "field": "notes.phrase",
        "direct_generator": [
          {
            "field": "notes.phrase",
            "suggest_mode": "always"
          },
          {
            "field": "notes.reverse",
            "suggest_mode": "always",
            "pre_filter": "reverse_analyzer",
            "post_filter": "reverse_analyzer"
          }
        ]
      }
    }
  }
}
```

* The bulk request must have a new line at the end
* The meta data has to be in one line
* Each document data has to be in one line
* Every document data has to be preceded by the meta data
* It should be noted that the body of a bulk request is not a valid JSON document. The individual lines are not separated by comma (,)

```
{ "create": { "_index": "mytestindex", "_id": "1"}} 
{ "name": "Saptarshi Basu", "name_completion": ["Saptarshi Basu", "Basu, Saptarshi", "Saptarshi", "Basu"], "notes": "When I hear you give your reason, I remarked, it always appears to me to be so ridiculously simple that I could always do it myself", "billgroup": "1110345265"}
{ "create": { "_index": "mytestindex", "_id": "2"}} 
{ "name": "Manasa Mahakud", "name_completion": ["Manasa Mahakud", "Mahakud, Manasa", "Manasa", "Mahakud"], "notes": "At each successive instance of your reasoning, I'm baffled until you explain the process and yet I believe my eyes are as good as yours", "billgroup": "1345281112"}
{ "create": { "_index": "mytestindex", "_id": "3"}} 
{ "name": "Sayantan Biswas",  "name_completion": ["Sayantan Biswas", "Biswas, Sayantan", "Sayantan", "Biswas"],  "notes": "Smile, an everlasting smile, a smile can bring a tear to me. You think that I don't even mean a single word I say. It's only words and words are all I have to take your hear away", "billgroup": "1110345265"}
{ "create": { "_index": "mytestindex", "_id": "4"}} 
{ "name": "Reshmi Raju", "name_completion": ["Reshmi Raju", "Raju, Reshmi", "Reshmi", "Raju"], "notes": "When the bough breaks, the craddle will fall and it has fallen here. Makers of man, creators of leader, be careful what kind of leaders you are producing here, because I see you are killing the very spirit this institution proclaims it instills", "billgroup": "1211346576"}

```

```
{
  "suggest": {
    "text" : "Sa",
    "my-suggest-1" : {
      "completion" : {
        "field" : "name_completion",
        "fuzzy": {
          "fuzziness": 1
        },
        "skip_duplicates": "true"
      }
    }
  }
}
```

```
POST /mytestindex/_search
{
  "query": {
    "match": {
      "notes.phrase": {
        "query": "smile a everlasting",
        "operator" : "and",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

## References

* https://www.elastic.co/guide/en/elasticsearch/resiliency/current/index.html
