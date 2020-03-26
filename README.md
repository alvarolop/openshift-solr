# Containerized SOLR On OpenShift

[Apache Solr](https://lucene.apache.org/solr/) is a scalable and fault tolerant search server backer by the [Lucene search library](https://lucene.apache.org/index.html). It provides distributed indexing, replication and load-balanced querying, automated failover and recovery, centralized configuration and more. This repository provides a starting point to deploy Solr in HA on Openshift.

An Apache Solr deployment consists on two steps:
- Deployment of a Zookeeper ensemble.
- Deployment of the Solr server.

## Official container images

The intention of this project is to keep aligned as much as possible with the official Docker images for Solr and Zookeeper. For that reason, the new images build in this project will have the minimum changes possible. These are the base images and its version:

- [Apache Solr 8.4.1](https://hub.docker.com/_/solr/).
- [Zookeeper 3.5.7](https://hub.docker.com/_/zookeeper).

NOTE: This project was only tested for the versions mentioned above. Other versions may require different configuration.

## Preparation

You may need to execute the following commands prior to the creation of the Zookeeper and Solr components. First, define the name of the OCP project where all the components will run:

```bash
export PROJECT_NAME=solr
```

After that, create the OCP project. Use the name and description that you prefer:
```bash
oc new-project $PROJECT_NAME --display-name="Solr" --description="Solr Project"
```


## Zookeeper ensemble

[ZooKeeper](https://zookeeper.apache.org/doc/r3.5.7/zookeeperOver.html) is a distributed, open-source coordination service for distributed applications. It exposes a simple set of primitives that distributed applications can build upon to implement higher level services for synchronization, configuration maintenance, and groups and naming. Like the distributed processes it coordinates, ZooKeeper itself is intended to be replicated over a sets of hosts. This is called an ensemble.

In order to deploy a simple Zookeeper ensemble, we will need, at least, several OCP objects: a Service, a headless Service, a StatefulSet, a Persistent Volume Claim and a ConfigMap. The structure followed for the Zookeeper OCP template is based on the [Zookeeper Helm chart](https://github.com/helm/charts/blob/master/incubator/zookeeper/README.md). 

By default, OpenShift Container Platform runs containers using [an arbitrarily assigned user ID](https://docs.openshift.com/container-platform/4.3/openshift_images/create-images.html#use-uid_create-images). This provides additional security against processes escaping the container due to a container engine vulnerability and thereby achieving escalated permissions on the host node. 

In order to provide access for this arbitrary user to all the configuration folders, it is necessary to use a Dockerfile to modify the permissions of the `/conf/` folder and generate a new custom image. This process is managed by a BuildConfig and an ImageStream.

### Deploy a Zookeeper ensemble

To deploy the zookeeper ensemble, just follow these steps:

```bash
oc process -f zookeeper/zookeeper-template-bc.yaml | oc apply -f - -n $PROJECT_NAME
oc process -f zookeeper/zookeeper-template-sts.yaml -p IMAGE_NAMESPACE=$PROJECT_NAME | oc apply -f - -n $PROJECT_NAME
```

### Testing Zookeeper ensemble

There are many ways of checking that the Zookeeper ensemble is formed properly. The basic method is to execute the command `zkServer.sh status` in all the pods. The output should show that the client port was found in all of them and that only one is the `leader` while the rest are in mode `follower`. Use the following command:

```bash
for i in {0..2} ; do echo "==>> Zookeeper ${i}"; oc exec zookeeper-${i} zkServer.sh status; echo ""; done
```

Another way to test zookeeper is by using it. The following commands create an entry and retrieve its value:

```bash
oc rsh zookeeper-0
  zkCli.sh
    create /hello world
    get /hello
    deleteall /hello
```

Finally, in order to test the ensemble functionality, an entry should be created in one pod and retrieved in a different one. The following code shows an example of this test:

```bash
oc rsh zookeeper-0
  zkCli.sh
    create /hello world
    stat /hello
    quit
  exit
oc rsh zookeeper-2
  zkCli.sh
    get /hello
    stat /hello
    deleteall /hello
    stat /hello
    quit
  exit
oc rsh zookeeper-0
  zkCli.sh
    get /hello
    stat /hello
    quit
  exit
```

## SolrCloud cluster

In order to create a Solr cluster, we are going to use the official image of Solr provided by the Solr community in [DockerHub](https://hub.docker.com/_/solr/). As we are testing a simple HA scenario, we do not need to customize the Solr container image. If necessary, this repository provides a BuildConfig [here](solr/solr-template-bc.yaml).


### Deploy a SolrCloud cluster

To deploy the SolrCloud cluster, just apply the following OCP template:

```bash
oc process -f solr/solr-template-sts.yaml -p APPLICATION_NAME=solr | oc apply -n ${PROJECT_NAME} -f -
```

### Using Solr

In this section, we are going to upload some configuration files and use them to test that it works.

In order to make this commands easier to use, please export your Solr url as a variable:
```bash
export SOLR_URL=<your_solr_url>
```

#### Upload the config files

If you want to use a custom config for your collection, you first need to upload it, and then refer to it by name when you create the collection. There are two main options. The first option is using the [solr command](https://lucene.apache.org/solr/guide/8_4/solr-control-script-reference.html#upload-a-configuration-set) and the easiest one is using the [configset API](https://lucene.apache.org/solr/guide/8_4/configsets-api.html#configsets-create).

In this example, we are going to use the [Upload operation](https://lucene.apache.org/solr/guide/8_4/configsets-api.html#configsets-upload) of the ConfigSets API.

```bash
export CONFIG_SET_NAME=alvaroConfigSet
export CONFIG_SET_PATH=solr/examples/default_alvaro.zip

curl -X POST --header "Content-Type:application/octet-stream" --data-binary @${CONFIG_SET_PATH} "http://${SOLR_URL}/solr/admin/configs?action=UPLOAD&name=${CONFIG_SET_NAME}"
```

Check the ConfigSets uploaded:
```bash
curl "http://${SOLR_URL}/api/cluster/configs?omitHeader=true"
```

You should see the following output:
```json
{
  "configSets": [
    "_default",
    "alvaroConfigSet"
  ]
}
```


#### Solr collection

There are many ways of creating collections in docker-solr. The easiest one is using [Solr Collection API](https://lucene.apache.org/solr/guide/8_4/collection-management.html#collection-management):

```bash
export COLLECTION_NAME=myCollection
curl "http://${SOLR_URL}/solr/admin/collections?action=CREATE&name=${COLLECTION_NAME}&numShards=1&collection.configName=${CONFIG_SET_NAME}"
```

Check your collection list:

```bash
curl "http://${SOLR_URL}/solr/admin/collections?action=LIST"
```

You should see the following output:
```json
{
  "responseHeader": {
    "status": 0,
    "QTime": 0
  },
  "collections": [
    "myCollection"
  ]
}
```


### Solr useful links and information

Here are some useful links to documentation and code examples:

- [How the image works](https://github.com/docker-solr/docker-solr/blob/master/README.md#how-the-image-works) (Readme of Solr-docker).
- [FAQ Solr-docker](https://github.com/docker-solr/docker-solr/blob/master/Docker-FAQ.md).
- [Docker-solr examples](https://github.com/docker-solr/docker-solr-examples).
- [Films data set](https://github.com/apache/lucene-solr/tree/master/solr/example/films) for testing.




## Limitations 

- Zookeeper: Probably the Zookeeper `zk-run.sh` init script should be included in the Dockerfile instead of having it in the ConfigMap. This file should not change.

- Zookeeper: Due to the custom `/conf/zoo.cfg` with the urls of each Zookeeper node, it is not possible to scale up or down the cluster without redeploying all the replicas. Could be possible to automate it setting it to "Redeploy" and using the Kubernetes API: https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/#without-using-a-proxy

- Solr: Currently, Solr nodes register themselves in ZK using their IP. As IP is ephemeral in pods, you may want to use the hostname variable. To achieve that, you may use the following configuration: https://lucene.apache.org/solr/guide/8_4/taking-solr-to-production.html#solr-hostname



## Extra: Useful tools

### nslookup container
Command to generate a pod to test connections and dns resolutions:

```bash
oc run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
```
