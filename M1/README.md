# Using Apache Drill M1 

## Preparation

(OPTIONAL) Download and install the [Lilith](http://lilithapp.com/) logging and access event viewer.

Make sure you have Java 7 installed:

    $ java -version
    java version "1.7.0_11"
    Java(TM) SE Runtime Environment (build 1.7.0_11-b21)
    Java HotSpot(TM) 64-Bit Server VM (build 23.6-b04, mixed mode)

Download the Apache Drill M1 binary release from [people.apache.org/~jacques/apache-drill-1.0.0-m1.rc3/](http://people.apache.org/~jacques/apache-drill-1.0.0-m1.rc3/apache-drill-1.0.0-m1-binary-release.tar.gz).


## Running Apache Drill in distributed mode

What follows are instructions how to run Apache Drill in distributed mode (in a local cluster).

### Install Zookeeper (ZK)

As of this [mailing list thread](http://mail-archives.apache.org/mod_mbox/incubator-drill-dev/201310.mbox/%3C93C17963-04F8-41B6-B2D1-F90473F9DB90%40gmail.com%3E)
Drill M1 doesn't launch ZK in local mode, hence needs an external ZK instance running. 

So, to achieve this, do the following to install and set up ZK:

* Download ZK from [zookeeper.apache.org/releases.html](http://zookeeper.apache.org/releases.html) in the version 3.4.3
* I assume in the following that $ZK_HOME is `/Users/mhausenblas2/bin/zookeeper-3.4.3`
* Create a config file `conf/apache-drill-zoo.cfg` that holds the Apache Drill specific settings. You can find the of this [ZK config file](https://raw.github.com/mhausenblas/apache-drill-sandbox/master/M1/conf/apache-drill-zoo.cfg) content in the `conf/` directory of this repo.

Then, launch ZK as shown below:

    $ bin/zkServer.sh start apache-drill-zoo.cfg
    JMX enabled by default
    Using config: /Users/mhausenblas2/bin/zookeeper-3.4.3/bin/../conf/apache-drill-zoo.cfg
    Starting zookeeper ... STARTED

### Launch 3-node local cluster

For a simple 3-node cluster, extract three copies of Apache Drill into three different dirs `xxx-node1 to xxx-node3` as so:

    [~/sandbox/apache-drill-1.0.0-m1-cluster] $ ls -al
    total 115080
    drwxr-xr-x   7 mhausenblas2  staff   238B 28 Oct 05:40 .
    drwxr-xr-x  22 mhausenblas2  staff   748B 28 Oct 05:40 ..
    -rw-r--r--@  1 mhausenblas2  staff    56M 10 Sep 14:10 apache-drill-1.0.0-m1-binary-release.tar.gz
    drwxr-xr-x@  7 mhausenblas2  staff   238B 28 Oct 05:42 apache-drill-1.0.0-m1-node1
    drwxr-xr-x@  7 mhausenblas2  staff   238B 28 Oct 05:40 apache-drill-1.0.0-m1-node2
    drwxr-xr-x@  7 mhausenblas2  staff   238B 28 Oct 05:40 apache-drill-1.0.0-m1-node3

For each `apache-drill-1.0.0-m1-cluster/apache-drill-1.0.0-m1-nodeX` launch the respective Drillbit as so:

    $ pwd
    /Users/mhausenblas2/sandbox/apache-drill-1.0.0-m1-cluster/apache-drill-1.0.0-m1-node1
    $ export DRILL_LOG_DIR=$PWD/log
    $ ./bin/drillbit.sh start

Check if all Drillbits are started:

    $ jps
    4590 Drillbit
    14036 Drillbit
    14823 Jps
    13133 ZooKeeperMain
    14816 Drillbit
    11465 QuorumPeerMain

So above you see the three Drillbits and the respective ZK jobs, as expected. Next check in ZK if all three nodes are registered:

    $ pwd
    /Users/mhausenblas2/bin/zookeeper-3.4.3/
    $ bin/zkCli.sh -server 127.0.0.1:2181
    ...
    [zk: 127.0.0.1:2181(CONNECTED) 1] get /drill/drillbits1
    cZxid = 0x3
    ctime = Mon Oct 28 05:33:31 GMT 2013
    mZxid = 0x3
    mtime = Mon Oct 28 05:33:31 GMT 2013
    pZxid = 0x19
    cversion = 11
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 0
    numChildren = 3
    
OK, we have `numChildren = 3` so looking good.
    

### Submit a physical query plan

Make sure the data is available. For each `apache-drill-1.0.0-m1-cluster/apache-drill-1.0.0-m1-nodeX` you'd have the following data files via `sample-data/` available:

* [`donuts.json`](https://raw.github.com/mhausenblas/apache-drill-sandbox/master/M1/data/donuts.json)
* `nation.parquet` and `region.parquet` are already shipped with M1

Also we need the physical query plan (in at least one of the `nodeX/sample-data/`):

* [`physical_json_scan_test1.json`](https://raw.github.com/mhausenblas/apache-drill-sandbox/master/M1/data/physical_json_scan_test1.json)
* [`parquet_scan_union_screen_physical.json`](https://raw.github.com/mhausenblas/apache-drill-sandbox/master/M1/data/parquet_scan_union_screen_physical.json)


And then submit the plan from one of the nodes (`node1` in my case):

    $ pwd
    /Users/mhausenblas2/sandbox/apache-drill-1.0.0-m1-cluster/apache-drill-1.0.0-m1-node1
  

#### Scan JSON doc

    $ bin/submit_plan -f sample-data/physical_json_scan_test1.json -t physical -zk 127.0.0.1:2181

#### Scan Parquet doc

    $ bin/submit_plan -f sample-data/parquet_scan_union_screen_physical.json -t physical -zk 127.0.0.1:2181

  
### Shutdown cluster

First `./bin/drillbit.sh stop` for each Drillbit, then `bin/zkServer.sh stop`.


## Interactive query on single node with sqlline

The following describes how to use the `sqlline` for single-node SQL queries.

    $ pwd
    /Users/mhausenblas2/sandbox/apache-drill-1.0.0-m1-cluster/apache-drill-1.0.0-m1-node1/

Make sure the following env vars are set:

    $ export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_11.jdk/Contents/Home
    $ export DRILL_LOG_DIR=$PWD/log
    $ ./bin/drillbit.sh start

    $ ./bin/sqlline -u jdbc:drill:schema=parquet-local

    0: jdbc:drill:schema=parquet-local>
    SELECT _MAP['R_REGIONKEY'] as region_key, _MAP['R_NAME'] AS name, _MAP['R_COMMENT'] AS comment FROM "sample-data/region.parquet";
    SELECT count(distinct _MAP['N_REGIONKEY']) FROM "sample-data/nation.parquet";	
    SELECT _MAP['N_REGIONKEY'] as regionKey, _MAP['N_NAME'] as name FROM "sample-data/nation.parquet" WHERE cast(_MAP['N_NAME'] as varchar) < 'M';

    $ ./bin/sqlline -u jdbc:drill:schema=json


More queries at on the [Wiki](https://cwiki.apache.org/confluence/display/DRILL/Demo+HowTo).
