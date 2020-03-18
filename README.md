# Containerized SOLR On OpenShift
Apache SOLR makes it easy to add search capability into your apps.  SOLR is a search server (backed by the Lucene serach library).  This repository provides a way for you to take advantage of that in OpenShift.

## There are 2 distinct parts to this repo:
   
### (1) A Dockerfile

Which overrides the [official Docker SOLR image][2] to tweak a few things in order to run SOLR efficiently on OpenShift.  

### (2) S2I scripts

Which allow you to easily push your project specific configuration files into the SOLR container.

## How to use all of this with your apps
Sounds cool right?  It is.  And here's how you can use it.

### Quick Start
* Bring up a local OpenShift cluster.
  * There are some [Chocolatey Scripts](https://github.com/WadeBarnes/dev-tools/tree/master/chocolatey) that make it very easy to do this on Windows.
* Run the `buildLocalProject.sh` script in the `openshift` directory.

This will create a **Solr** project and generate all of the build and deployment configurations needed for a working Solr instance complete with a core configured from source.

### If you just want to try running SOLR in OpenShift...

Create the SOLR app from a Docker image
`> oc new-app dudash/openshift-solr --name=solr-imageonly-demo`

Now you can access it via the route that was automatically exposed on port 8983 and whereever your OpenShift apps route (e.g. openshift-solr-myproject.127.0.0.1.nip.io).  Note: this won't autogenerate a SOLR core.

### If you want to provide configuration in an automated way...
* Create a repo
* Create a folder called `solr/autocore/conf` and add SOLR config files for your desired SOLR configuration
  (Refer to the sample in the project).
* Wire up your OpenShift build configuration to point to the repo with configuration.  Refer to the script and template samples in the `openshift` directory.

### The S2I process
This repo doesn't require [the s2i tool](https://github.com/openshift/source-to-image) to build the image.  However, if you look into the Dockerfile, it does set some s2i LABELS in order for OpenShift to be able to use it as an s2i builder image.

#### assemble script
Refer to the documentation in the script for details.

#### run script
Refer to the documentation in the script for details.

## Want to help?
If you find and issues, go ahead and write them up.  If you want to submit some code changes, please see the [CONTRIBUTING][3] docs.


[1]: https://github.com/docker-solr/docker-solr
[2]: https://store.docker.com/images/f4e3929d-d8bc-491e-860c-310d3f40fff2?tab=description
[3]: ./CONTRIBUTING.md




## Alvaro Films

https://github.com/apache/lucene-solr/tree/master/solr/example/films


## Create OCP resources


### 0. Set up variables

```bash
export PROJECT_NAME=solr
```




### 1. Create OCP project

```bash
oc new-project $PROJECT_NAME --display-name="Solr" --description="Solr Test Project"
```


### 2. Process OCP templates

```bash
oc process -f openshift/solr-template-bc-base.yaml | oc apply -n ${PROJECT_NAME} -f -

oc process -f openshift/solr-template-bc-s2i.yaml | oc apply -n ${PROJECT_NAME} -f -

oc process -f openshift/solr-template-ss.yaml -p APPLICATION_NAME=solr | oc apply -n ${PROJECT_NAME} -f -
```


### 3. Index data

```bash
curl http://solr-solr.alvarolop.lab.upshift.rdu2.redhat.com/solr/films/schema -X POST -H 'Content-type:application/json' --data-binary '{
    "add-field" : {
        "name":"name",
        "type":"text_general",
        "multiValued":false,
        "stored":true
    },
    "add-field" : {
        "name":"initial_release_date",
        "type":"pdate",
        "stored":true
    }
  }'
```


## Extra: Build the image manually

```bash
#!/bin/bash
SCRIPT_DIR=$(dirname $0)

docker build -t 'openshift-solr' -f ${SCRIPT_DIR}/Dockerfile ${SCRIPT_DIR}
```
 


## External zookeeper ensemble



### Generate Zookeeper custom image

```bash
oc process -f openshift/zookeeper-template-bc.yaml | oc apply -f - -n $PROJECT_NAME
```


### Deploy application

```bash
<!-- oc import-image zookeeper:3.5.7 --from=zookeeper:3.5.7 -n $PROJECT_NAME --confirm -->
oc process -f openshift/zookeeper-template-ss.yaml | oc apply -f - -n $PROJECT_NAME
```










## Useful commands and pods


### nslookup pod
Useful command to generate a pod to test connection

```bash
oc run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
```

### Check Zookeeper ensemble status

```bash
for i in {0..2} ; do echo "==>> Zookeeper ${i}"; oc exec zookeeper-${i}-0 zkServer.sh status; echo ""; done
```


### Testing Zookeeper ensemble with put and get

Test 1
```bash
oc rsh zookeeper-0-0
  zkCli.sh
  create /hello world
  get /hello
  deleteall /hello
```

Test 2
```bash
oc rsh zookeeper-0-0
  zkCli.sh
    create /hello world
    stat /hello
    quit
  exit
oc rsh zookeeper-2-0
  zkCli.sh
    get /hello
    stat /hello
    deleteall /hello
    stat /hello
    quit
  exit
oc rsh zookeeper-0-0
  zkCli.sh
    get /hello
    stat /hello
    quit
  exit
```



