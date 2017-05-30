# Natively Build Spark on Kubernetes
This tutorial leverages the framework built in this fork of spark [here](https://github.com/apache-spark-on-k8s/spark) which corresponds to an umbrella Spark JIRA issue focused on [here](https://issues.apache.org/jira/browse/SPARK-18278).

First you will need to build the most recent version of spark (with Kubernetes support). This can be done with the following:

```bash
$ git clone https://github.com/ifilonenko/spark.git
$ cd spark
$ build/mvn compile -Pkubernetes -pl resource-managers/kubernetes/core -am -DskipTests
$ dev/make-distribution.sh --tgz -Phadoop-2.7 -Pkubernetes
```

You should now have a spark distribution with Kubernetes support with versioning of the form: `spark-2.1.0-k8s-0.1.0-SNAPSHOT-bin-2.7.3.tgz.` The version that we are building with is Spark 2.1 and Hadoop 2.7.3. Next step is to ensure that you have `kubectl` and `docker` setup. 

Install `minikube`,`kubectl`, and `docker`. 

Check that these are properly installed by running:

```bash
$ minikube status
minikubeVM: Does Not Exist
localkube: N/A
$ kubectl help
...
$ docker help
...
```

At this point in time we need to start up the kubernetes cluster (in this tutorial locally):

```bash
$ minikube start --insecure-registry=localhost:5000; eval $(minikube docker-env)
$ kubectl cluster-info
Starting local Kubernetes v1.6.0 cluster...
Starting VM...
SSH-ing files into VM...
Setting up certs...
Starting cluster components...
Connecting to cluster...
Setting up kubeconfig...
Kubectl is now configured to use the cluster.
$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

You are now ready to begin. Unzip the distriubtion file into a directory and now you may start running spark jobs in your local minikube cluster. 

```bash
$ tar -xvf spark-2.1.0-k8s-0.1.0-SNAPSHOT-bin-2.7.3.tgz
$ cd spark-2.1.0-k8s-0.1.0-SNAPSHOT-bin-2.7.3
$ bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://https://192.168.99.100:8443 \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.submission.waitAppCompletion=true \
  --conf spark.kubernetes.driver.docker.image=ifilonenko/spark-driver:latest \
  --conf spark.kubernetes.executor.docker.image=ifilonenko/spark-executor:latest \
  --conf spark.kubernetes.initcontainer.docker.image=ifilonenko/spark-init:latest \
  local:///opt/spark/examples/jars/spark-examples_2.11-2.1.0-k8s-0.1.0-SNAPSHOT.jar 10000
``` 
You may now monitor the logs like this: 

```bash
$ kubectl get pods --watch
NAME                                             READY     STATUS    RESTARTS   AGE
spark-resource-staging-server-1303329619-72kp4   1/1       Running   0          1m
spark-pi-1496166532391   0/1       Pending   0         0s
spark-pi-1496166532391   0/1       Pending   0         0s
spark-pi-1496166532391   0/1       Init:0/1   0         0s
spark-pi-1496166532391   0/1       Init:0/1   0         13s
spark-pi-1496166532391   0/1       PodInitializing   0         15s
spark-pi-1496166532391   1/1       Running   0         19s
spark-pi-1496166532391-exec-1   0/1       Pending   0         0s
spark-pi-1496166532391-exec-1   0/1       Pending   0         0s
spark-pi-1496166532391-exec-1   0/1       Init:0/1   0         0s
spark-pi-1496166532391-exec-2   0/1       Pending   0         0s
spark-pi-1496166532391-exec-2   0/1       Pending   0         0s
spark-pi-1496166532391-exec-3   0/1       Pending   0         0s
spark-pi-1496166532391-exec-3   0/1       Pending   0         0s
spark-pi-1496166532391-exec-4   0/1       Pending   0         0s
spark-pi-1496166532391-exec-4   0/1       Pending   0         0s
spark-pi-1496166532391-exec-5   0/1       Pending   0         0s
spark-pi-1496166532391-exec-5   0/1       Pending   0         0s
spark-pi-1496166532391-exec-1   0/1       Init:0/1   0         1s
spark-pi-1496166532391-exec-1   0/1       PodInitializing   0         4s
...
$

Now you can watch logs with this function which follows the driver:

```bash
$ kubectl logs spark-pi-1496166532391 -f
...
```

If we are to pass in values using the dependency manager you can do it with the following:

Download the file linked in this repo called `kubernetes-resource-staging-server.yaml.`

Then run the following:

```bash
$ kubectl create -f kubernetes-resource-staging-server.yaml
$ kubect get services
NAME                             CLUSTER-IP   EXTERNAL-IP   PORT(S)           AGE
kubernetes                       10.0.0.1     <none>        443/TCP           7m
spark-resource-staging-service   10.0.0.42    <nodes>       10000:31000/TCP   6s
$ kubectl get pods
NAME                                             READY     STATUS    RESTARTS   AGE
spark-resource-staging-server-1303329619-72kp4   1/1       Running   0          24s
$ bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://https://192.168.99.100:8443 \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.submission.waitAppCompletion=true \
  --conf spark.kubernetes.driver.docker.image=ifilonenko/spark-driver:latest \
  --conf spark.kubernetes.executor.docker.image=ifilonenko/spark-executor:latest \
  --conf spark.kubernetes.initcontainer.docker.image=ifilonenko/spark-init:latest \
  --conf spark.kubernetes.resourceStagingServer.uri=http://192.168.99.100:31000 \
  examples/jars/spark-examples_2.11-2.1.0-k8s-0.1.0-SNAPSHOT.jar 10000
```

Logs can be following similairly but now you can specify the jar's locally by leveraging the resource-staging-server.