# Jaeger Spark dependencies
This is a Spark job that collects spans from storage, analyze links between services,
and stores them for later presentation in the UI. Note that it is needed for the production deployment.
`all-in-one` distribution does not need this job.

This job parses all traces on a given day, based on UTC. By default, it processes the current day,
but other days can be explicitly specified.

This repository is based on [zipkin-dependencies](https://github.com/openzipkin/zipkin-dependencies).

## Running Job Locally in IntelliJ
 
1. Install Latest Java 8 SDK
2. Make sure SDK is set in `File -> Project Structure`
  ![img_2.png](img_2.png)
3. Add TLS Cert to keystore.jks (cert may already be included)
    1. Port Forward to Elastic Search Instance :`kubectl --context [context] -n tracing get secrets tracing-es-elastic-user -o json | jq -r '.data.elastic' | base64 -d && echo`
    2. Download Certificate : `certigo connect localhost:9200 -j | jq -r '.certificates[0].pem' >> es.cert`
    3. Add Certificate (password is: `password` ) : `keytool -importcert -file es.cert -keystore keystore.jks -alias "ES-CERT-FOO"`
7. Get the Elastic Search (ES) password
`kubectl --context [context] -n tracing get secrets tracing-es-elastic-user -o json | jq -r '.data.elastic' | base64 -d && echo`
8. Add ES Password to Run Config
    1. Open Run Config Here and Click `Open Run/Debug Configurations`
       ![img.png](img.png)
    2. Set ES_PASSWORD
       ![img_1.png](img_1.png)
10. Port-Forward to ES : `kubectl --context [context] -n tracing port-forward svc/tracing-es-http 9200:9200`
11. Run or Debug Job

![img_3.png](img_3.png)
    
## Quick-start
Spark job can be run as docker container and also as java executable:

Docker:
```bash
$ docker run --env STORAGE=cassandra --env CASSANDRA_CONTACT_POINTS=host1,host2 jaegertracing/spark-dependencies
```

Use `--env JAVA_OPTS=-Djavax.net.ssl.` to set trust store and other Java properties.

As jar file:
```bash
STORAGE=cassandra java -jar jaeger-spark-dependencies.jar
```

## Usage
By default, this job parses all traces since midnight UTC. You can parse traces for a different day
via an argument in YYYY-mm-dd format, like 2016-07-16 or specify the date via an env property.

```bash
# ex to run the job to process yesterday's traces on OS/X
$ STORAGE=cassandra java -jar jaeger-spark-dependencies.jar `date -uv-1d +%F`
# or on Linux
$ STORAGE=cassandra java -jar jaeger-spark-dependencies.jar `date -u -d '1 day ago' +%F`
```

### Configuration
`jaeger-spark-dependencies` applies configuration parameters through environment variables.

The following variables are common to all storage layers:

    * `SPARK_MASTER`: Spark master to submit the job to; Defaults to `local[*]`
    * `DATE`: Date in YYYY-mm-dd format. Denotes a day for which dependency links will be created.

### Cassandra
Cassandra is used when `STORAGE=cassandra`.

    * `CASSANDRA_KEYSPACE`: The keyspace to use. Defaults to "jaeger_v1_dc1".
    * `CASSANDRA_CONTACT_POINTS`: Comma separated list of hosts / ip addresses part of Cassandra cluster. Defaults to localhost
    * `CASSANDRA_LOCAL_DC`: The local DC to connect to (other nodes will be ignored)
    * `CASSANDRA_USERNAME` and `CASSANDRA_PASSWORD`: Cassandra authentication. Will throw an exception on startup if authentication fails
    * `CASSANDRA_USE_SSL`: Requires `javax.net.ssl.trustStore` and `javax.net.ssl.trustStorePassword`, defaults to false.
    * `CASSANDRA_CLIENT_AUTH_ENABLED`: If set enables client authentication on SSL connections. Requires `javax.net.ssl.keyStore` and `javax.net.ssl.keyStorePassword`, defaults to false.

Example usage:

```bash
$ STORAGE=cassandra CASSANDRA_CONTACT_POINTS=localhost:9042 java -jar jaeger-spark-dependencies.jar
```
### Elasticsearch
Elasticsearch is used when `STORAGE=elasticsearch`.

    * `ES_NODES`: A comma separated list of elasticsearch hosts advertising http. Defaults to
                  localhost. Add port section if not listening on port 9200. Only one of these hosts
                  needs to be available to fetch the remaining nodes in the cluster. It is
                  recommended to set this to all the master nodes of the cluster. Use url format for
                  SSL. For example, "https://yourhost:8888"
    * `ES_NODES_WAN_ONLY`: Set to true to only use the values set in ES_HOSTS, for example if your
                           elasticsearch cluster is in Docker. If you're using a cloudprovider
                           such as AWS Elasticsearch, set this to true. Defaults to false
    * `ES_USERNAME` and `ES_PASSWORD`: Elasticsearch basic authentication. Use when X-Pack security
                                       (formerly Shield) is in place. By default no username or
                                       password is provided to elasticsearch.
    * `ES_CLIENT_NODE_ONLY`: Set to true to disable elasticsearch cluster nodes.discovery and enable nodes.client.only.
                             If your elasticsearch cluster's data nodes only listen on loopback ip, set this to true.
                             Defaults to false
    * `ES_INDEX_PREFIX`: index prefix of Jaeger indices. By default unset.
    * `ES_TIME_RANGE`: How far in the past the job should look to for spans, the maximum and default is `24h`.
                       Any value accepted by [date-math](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math) can be used here, but the anchor is always `now`.

Example usage:

```bash
$ STORAGE=elasticsearch ES_NODES=http://localhost:9200 java -jar jaeger-spark-dependencies.jar
```

## Building locally
To build the job locally and run tests:
```bash
./mvnw clean install # if failed add SPARK_LOCAL_IP=127.0.0.1
STORAGE=elasticsearch ES_NODES=http://localhost:9200 java -jar jaeger-spark-dependencies/target/jaeger-spark-dependencies-0.0.1-SNAPSHOT.jar
docker build -t jaegertracing/spark-dependencies:latest .
```

In tests it's possible to specify version of Jaeger images by env variable `JAEGER_VERSION`
or system property `jaeger.version`. By default tests are using latest images.

## License

[Apache 2.0 License](./LICENSE).
