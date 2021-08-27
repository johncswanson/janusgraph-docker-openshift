# Modfify Existing JansuGraph Docker to work with OpenShift

The JanusGraph Docker image from the official repo deploys fine into Kubernetes but runs into errors when deployed into OpenShift. There are few things that need to be modified before you can deploy:

* Fork the repo `https://github.com/JanusGraph/janusgraph-docker`.
* [Change the file and group ownership](https://developer.ibm.com/learningpaths/universal-application-image/design-universal-image/#6-set-group-ownership-and-file-permission) to root (0) for related folders. The following modifications apply to the `Dockerfile`:
```bash
chgrp -R 1001:0 ${JANUS_HOME} ${JANUS_INITDB_DIR} ${JANUS_CONFIG_DIR} ${JANUS_DATA_DIR} && \
chmod -R g+w ${JANUS_HOME} ${JANUS_INITDB_DIR} ${JANUS_CONFIG_DIR} ${JANUS_DATA_DIR}

RUN chmod u+x /opt/janusgraph/bin/gremlin.sh
RUN chmod u+x /opt/janusgraph/conf/remote.yaml
```
* Change the `JANUS_PROPS_TEMPLATE` value to `cql` as you will be using Cassandra as the back end.
* Since you will only be using the latest version, change the version to the latest in `build-images.sh`. You will create a copy of that file to `build-images-ibm.sh` and modify it there. These modifications include commenting out a few lines. The following modifications are applied to the build script:

```bash
# optional version argument
version="${1:-}"
# get all versions
# versions=($(ls -d [0-9]*))
# get the last element of sorted version folders
# latest_version="${versions[${#versions[@]}-1]}"

# override to run the latest version only:
versions="0.5"
latest_version="0.5"
```
* Create `janusgraph-cql-server.properties` in the latest version directory (which in this case is `0.5`) and add the following properties:

```bash
gremlin.graph=org.janusgraph.core.JanusGraphFactory
storage.backend=cql
storage.hostname=cassandra-service
storage.username=cassandra
storage.password=cassandra
storage.cql.keyspace=janusgraph
storage.cassandra.replication-factor=3
storage.cassandra.replication-strategy-class=org.apache.cassandra.locator.NetworkTopologyStrategy
cache.db-cache = true
cache.db-cache-clean-wait = 20
cache.db-cache-time = 180000
cache.db-cache-size = 0.5
storage.directory=/var/lib/janusgraph/data
index.search.backend=lucene
index.search.directory=/var/lib/janusgraph/index
```

These are properties that allows JanusGraph to talk to Cassandra as Cassandra will be storing the data in a distributed fashion.

After these changes, make sure to update `janusgraph-cql-server.properties` with the `cluster-ip` of the Cassandra service. Update `storage.hostname` with the `Cluster-IP`.

![Cluster IP](../images/cluster-ip.png)

Now you can build and deploy the JanusGraph Docker image to OpenShift by running the following command.
> NOTE: modify the docker image name in the script: `IMAGE_NAME="<your docker image repository>"`

```bash
$ ./build-images-ibm.sh -- if you have created a new file
```

or...

```bash
$ ./build-images.sh -- if you have modified the file provided by JanusGraph
```