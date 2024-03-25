- [Prerequisite - OpenShift Access](#prerequisite---openshift-access)
- [Install CFK Operator](#install-cfk-operator)
  - [Operator Hub (Internal Connected)](#operator-hub-internal-connected)
  - [Internet Connection (with Helm)](#internet-connection-with-helm)
  - [Airgapped (with Helm)](#airgapped-with-helm)
    - [Spin up Your Own Docker Registry](#spin-up-your-own-docker-registry)
- [Generate CA for TLS](#generate-ca-for-tls)
- [Install Confluent Platform Cluster](#install-confluent-platform-cluster)
  - [Simple No-TLS Deployment without external access with manual Control Center route](#simple-no-tls-deployment-without-external-access-with-manual-control-center-route)
  - [Simple AutoGenerated TLS Deployment with external access with generated ControlCenter route](#simple-autogenerated-tls-deployment-with-external-access-with-generated-controlcenter-route)
  - [Simple AutoGenerated TLS Deployment with RBAC / LDAP](#simple-autogenerated-tls-deployment-with-rbac--ldap)
- [General Tips and Tricks](#general-tips-and-tricks)
  - [Checking the geneerated properties in each pod](#checking-the-geneerated-properties-in-each-pod)
  - [Test connectivity](#test-connectivity)
    - [Brokers](#brokers)
    - [Zookeeper](#zookeeper)
    - [Working with ZK and Broker CLI tools within the POD](#working-with-zk-and-broker-cli-tools-within-the-pod)
    - [Working with ZK from outside of the pod](#working-with-zk-from-outside-of-the-pod)
  - [Working with Routes](#working-with-routes)


## Prerequisite - OpenShift Access

If you already have access to OpenShift cluster and relevant CLI tools, log into it with kubeadmin level privileges with `oc login`

or

If you need an OCP cluster, an easy way is to run locally with `crc`. Create/login into RedHat OpenShift portal here https://console.redhat.com/openshift/create/local and download `OpenShift Local`, which is `crc` and also the `oc` command line tool from here https://console.redhat.com/openshift/downloads. 

The following configs I found to work with locally spinning up CFK with single node configurations.



Navigate to the above, create 

```
# one time setting (unless you crc delete the VM)
crc setup
crc config set memory 16384
crc config set disk-size 200

# start local OCP cluster
crc start

eval $(crc oc-env)
oc login -u kubeadmin https://api.crc.testing:6443

# open up UI
crc console

# stop local OCP cluster, and if you forget, 
crc stop
```


## Install CFK Operator

### Operator Hub (Internal Connected)

Multiple ways of doing this, but the simpliest way is through the UI (https://console-openshift-console.apps-crc.testing/operatorhub if running crc locally)


Navigate to Administrator -> OperatorHub -> Search for Confluent -> and press install (by default it will install for all namespaces)


### Internet Connection (with Helm)
```
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update

oc new-project confluent

# namespaced - operator is installed in confluent namespace and wes specify namespaces for the operator to watch
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes \
--set podSecurity.enabled=false --namespace confluent \
--set namespaceList="{confluent,cfk,mrc-kafka,mrc-broker}" \ 
--set namespaced=true

# not namespaced - operator is installed in confluent namespace but will watch the all namespaces
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes --set podSecurity.enabled=false --namespace confluent --set namespaced=false
```


### Airgapped (with Helm)

On your jump host, install docker, helm, oc, and other CLI tools.

```
# copy to your work directory and untar
curl -O https://confluent-for-kubernetes.s3-us-west-1.amazonaws.com/confluent-for-kubernetes-2.8.0.tar.gz
tar zxvf confluent-for-kubernetes-2.8.0.tar.gz

export CFK_NAMESPACE=confluent
export REGISTRY_BASE=<PRIVATE_DOCKER_REGISTRY>

# Tag and push if this image isn't at your registry yet
# docker pull docker.io/confluentinc/confluent-operator:0.921.2
# docker tag docker.io/confluentinc/confluent-operator:0.921.2 $REGISTRY_BASE/confluentinc/confluent-operator:0.921.2
# docker push $REGISTRY_BASE/confluentinc/confluent-operator:0.921.2

# create namespace for confluent
oc new-project $CFK_NAMESPACE

# we can either use values.yaml to set these, or directly in the helm command

# Point to your private registry as well as allow operator to run with default SCC
helm upgrade --install confluent-operator \
  ./confluent-for-kubernetes-2.8.0-20240214/helm/confluent-for-kubernetes \
  --namespace=$CFK_NAMESPACE \
  --set podSecurity.enabled=false \
  --set image.registry=$REGISTRY_BASE
```

#### Spin up Your Own Docker Registry
If you need a docker registry for pushing your own images, you can host one yourself. First have docker installed on your VM (you can technically run this on your local machine as well, but it would take up a lot of storage)

```
# First install docker for your OS
open https://docs.docker.com/engine/install/

# add the following lines to the daemon.json to allow insecure registry (this is the file location for Linux OS, might be different for other OS)
sudo vim /etc/docker/daemon.json
{
    "insecure-registries" : [ "<hostname>:5000" ]
}

# START DOCKER REGISTRY LOCALLY
sudo docker run -d -p 5000:5000 --name registry registry:2.7

# or restart when the host is restarted 
sudo docker restart registry
```


## Generate CA for TLS

If we are using the autogenerated TLS option included with CFK, it will require a CA, and we are generating one here

```
mkdir -p ./certs

openssl genrsa -out ./certs/ca-key.pem 2048

openssl req -new -key ./certs/ca-key.pem -x509 \
  -days 1000 \
  -out ./certs/ca.pem \
  -subj "/C=US/ST=CA/L=MountainView/O=Confluent/OU=Operator/CN=TestCA"

# note we're working off of "confluent" namespace
oc create secret tls ca-pair-sslcerts \
  --cert=./certs/ca.pem \
  --key=./certs/ca-key.pem -n confluent
```

## Install Confluent Platform Cluster

### Simple No-TLS Deployment without external access with manual Control Center route

If we are wanting to deploy a simple cluster with no TLS (excluding the manaully created Route), use this option for a quick cluster test. Note the route YAML and the overrides needed for the single broker cluster
```
# deploy a simple CFK cluster with no TLS, with a manually created route for easy
# Control Center Access
oc apply -f ./simple-single-node-ocp/ocp-single-node-basic-no-tls.yaml

# Navigate to manually created route for Control Center UI https://cc-confluent.apps-crc.testing/

# cleanup
oc delete -f ./simple-single-node-ocp/ocp-single-node-basic-no-tls.yaml
```

### Simple AutoGenerated TLS Deployment with external access with generated ControlCenter route

```
oc apply -f ./simple-single-node-ocp/ocp-single-node-basic-route-tls.yaml

# Navigate to created route for Control Center UI https://controlcenter.apps-crc.testing/

oc delete -f ./simple-single-node-ocp/ocp-single-node-basic-route-tls.yaml
```

### Simple AutoGenerated TLS Deployment with RBAC / LDAP

```
export TUTORIAL_HOME=~/github/confluent-kubernetes-examples/security/production-secure-deploy-ldap-rbac-all

# sor zookeeper
oc create secret generic zookeeper-jaas-digest \
  --from-file=digest-users.json=./single-node-ocp-ldap-rbac/zookeeper-sasl-jaasconfig-user.json \
  --namespace confluent

oc create secret generic kafka-zookeeper-digest \
  --from-file=digest.txt=./single-node-ocp-ldap-rbac/zookeeper-kafka-jaas-digest.txt \
  --namespace confluent

# MDS Token
oc create secret generic mds-token \
  --from-file=mdsPublicKey.pem=$TUTORIAL_HOME/../../assets/certs/mds-publickey.txt \
  --from-file=mdsTokenKeyPair.pem=$TUTORIAL_HOME/../../assets/certs/mds-tokenkeypair.txt \
  --namespace confluent

# create secret which is referenced by MDS for bind - expected key is "ldap.txt"
oc create secret generic mds-credential-auth \
  --from-file=ldap.txt=./single-node-ocp-ldap-rbac/ldap-search-bind.txt \
  --namespace confluent

oc create secret generic kafka-admin-bearer \
  --from-file=bearer.txt=./single-node-ocp-ldap-rbac/kafka-admin-bearer.txt \
  --namespace confluent

oc create secret generic kafka-interbroker-cred \
  --from-file=plain-interbroker.txt=./single-node-ocp-ldap-rbac/kafka-admin-interbroker.txt \
  --namespace confluent

oc create secret generic kafka-listener-external-cred \
  --from-file=plain.txt=./single-node-ocp-ldap-rbac/kafka-admin-external.txt \
  --namespace confluent


# Control center to kafka creds
# key can either be plain.txt or controlcenter-client-plain.txt
oc create secret generic controlcenter-client-creds \
  --from-file=plain.txt=./single-node-ocp-ldap-rbac/kafka-admin-external.txt \
  --namespace confluent



# Apply Yaml
oc apply -f ./single-node-ocp-ldap-rbac/ocp-single-node-basic-route-tls-rbac-ldap.yaml


# Navigate to created route for Control Center UI https://controlcenter.apps-crc.testing/
open https://controlcenter.apps-crc.testing/



# Clean up
oc delete -f ./single-node-ocp-ldap-rbac/ocp-single-node-basic-route-tls-rbac-ldap.yaml

oc delete secret zookeeper-jaas-digest
oc delete secret kafka-zookeeper-digest
oc delete secret mds-token
oc delete secret mds-credential-auth
oc delete secret kafka-admin-bearer
oc delete secret kafka-interbroker-cred
oc delete secret kafka-listener-external-cred
oc delete secret controlcenter-client-creds
```


## General Tips and Tricks

### Checking the geneerated properties in each pod
If you wanted to cat out the properties file that is generated by CFK by each of the component pods, one way is to exec into the pod and run cat directly

```
oc exec kafka-0 -- cat /opt/confluentinc/etc/kafka/kafka.properties
oc exec kafka-0 -- cat /opt/confluentinc/etc/kafka/jvm.properties
oc exec kafka-0 -- cat /opt/confluentinc/etc/kafka/log4j.properties

# zk 
oc exec zookeeper-0 -- cat /opt/confluentinc/etc/zookeeper/zookeeper.properties
```

Or exec into the pod to look around
```
oc rsh kafka-0
cd /opt/confluentinc/etc/kafka
```

Different components will have different directory names, as shown with zookeeper and kafka above. Replace with appropricate values depending on which component you're working with.


### Test connectivity

Download kcat if it's not available on your machine

#### Brokers

Checking from external to the cluster route

```
BOOTSTRAP_SERVER='kafka.apps-crc.testing:443'

# SSL enabled listner with SSL only and no authn/authz (when we use the default route generated options from CFK, TLS is required)
kcat -b $BOOTSTRAP_SERVER -X security.protocol=SSL -X enable.ssl.certificate.verification=false -L

or

# SSL enabled listner with SASL_PLAIN
kcat -b $BOOTSTRAP_SERVER \
            -X security.protocol=SASL_SSL \
            -X sasl.mechanisms=PLAIN \
            -X sasl.username=$KAFKA_KEY \
            -X sasl.password=$KAFKA_SECRET \
            -L
```

#### Zookeeper

Internally from the pod

```
oc rsh zookeeper-0
zookeeper-shell localhost:2181 # or from another pod, use "zookeeper.confluent.svc.cluster.local:2181/kafka-confluent"

ls -s /kafka-confluent/brokers/ids

```

Externally from route or external methods
```
zookeeper-shell zookeeper.apps-crc.testing:443

# TLS 
zookeeper-shell zookeeper.apps-crc.testing:443 -zk-tls-config-file ./zk-connect.config
```


#### Working with ZK and Broker CLI tools within the POD
```
oc rsh zookeeper-0
zookeeper-shell localhost:2181
ls -s /
```

#### Working with ZK from outside of the pod

If you want to test connection to ZK outside the pod but you are using autogenerated certs (which doesn't include external domains), you can still do so by skipping hostname verification when connecting with zookeeper-shell

```
# steal the generated truststore so you don't have to generate it yourself
oc cp confluent/kafka-0:/mnt/sslcerts/..data/jksPassword.txt ./jksPassword.txt
oc cp confluent/kafka-0:/mnt/sslcerts/..data/truststore.jks ./truststore.jks


cat <<EOT > zk-connect.config
# Connect to the ZooKeeper port configured for TLS
zookeeper.connect=zk1:2182,zk2:2182,zk3:2182
# Required to use TLS-to-ZooKeeper (default is false)
zookeeper.ssl.client.enable=true
# Required to use TLS-to-ZooKeeper
zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
# Define trust store to use encryption-only TLS-to-ZooKeeper; ignored unless zookeeper.ssl.client.enable=true
# There is no need to set keystore information assuming ssl.clientAuth=none on ZooKeeper
zookeeper.ssl.truststore.location=./truststore.jks
zookeeper.ssl.truststore.password=mystorepassword
EOT
```

### Working with Routes

If the default route options that are generated by CFK custom resources doesn's suit your needs, it's possible to generate your own routes as long as you configure them to point to the internal services correctly.

For example, a manual route is created for the no-tls example above because CFK doesn't support natively generating routes without TLS enabled. Creating a route manually with TLS termination provides UI access to Control Center for easy access during POC stage without depending on what CFK requires us to do.