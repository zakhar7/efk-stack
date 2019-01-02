REF: https://www.digitalocean.com/community/tutorials/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes

# Step 1 — Creating a Namespace
    kubectl create -f 0-namespace.yaml

Display the namespaces

    kubectl get namespaces

# Step 2 — Creating the Elasticsearch
## ES Headless Service
    kubectl create -f 1-elasticsearch_svc.yaml

Display

    kubectl get services --namespace=kube-logging

## ES StatefulSet
The -oss suffix ensures that we use the open-source version of Elasticsearch. If you'd like to use the default version containing X-Pack (which includes a free license), omit the -oss suffix.

We define a StatefulSet called es-cluster in the kube-logging namespace. We then associate it with our previously created elasticsearch Service using the serviceName field. This ensures that each Pod in the StatefulSet will be accessible using the following DNS address: es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local, where [0,1,2] corresponds to the Pod's assigned integer ordinal.

We specify 3 replicas (Pods) and set the matchLabels selector to app: elasticseach, which we then mirror in the .spec.template.metadata section. The .spec.selector.matchLabels and .spec.template.metadata.labels fields must match.

We then use the resources field to specify that the container needs at least 0.1 vCPU guaranteed to it, and can burst up to 1 vCPU (which limits the Pod's resource usage when performing an initial large ingest or dealing with a load spike).
We then open and name ports 9200 and 9300 for REST API and inter-node communication, respectively. We specify a volumeMount called data that will mount the PersistentVolume named data to the container at the path 

    /usr/share/elasticsearch/data.

Set the environment variables:
* cluster.name: The Elasticsearch cluster's name, which in this guide is k8s-logs.
node.name: The node's name, which we set to the .metadata.name field using valueFrom. This will resolve to es-cluster-[0,1,2], depending on the node's assigned ordinal.
* discovery.zen.ping.unicast.hosts: This field sets the discovery method used to connect nodes to each other within an Elasticsearch cluster. We use unicast discovery, which specifies a static list of hosts for our cluster. In this guide, thanks to the headless service we configured earlier, our Pods have domains of the form es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local, so we set this variable accordingly. Using local namespace Kubernetes DNS resolution, we can shorten this to es-cluster-[0,1,2].elasticsearch. 
* discovery.zen.minimum_master_nodes: We set this to (N/2) + 1, where N is the number of master-eligible nodes in our cluster. In this guide we have 3 Elasticsearch nodes, so we set this value to 2 (rounding down to the nearest integer).
* ES_JAVA_OPTS: Here we set this to -Xms512m -Xmx512m which tells the JVM to use a minimum and maximum heap size of 512 MB. You should tune these parameters depending on your cluster's resource availability and needs.

Define several Init Containers that run before the main elasticsearch app container. These Init Containers each run to completion in the order they are defined.

"volumeClaimTemplates" Kubernetes will use this to create PersistentVolumes for the Pods. In the block above, we name it "data" (which is the name we refer to in the volumeMounts defined previously), and give it the same "app: elasticsearch" label as our StatefulSet.

We then specify its access mode as ReadWriteOnce, which means that it can only be mounted as read-write by a single node. We define the staorage class as "standard" for the minikube only. You should change this value depending on where you are running your Kubernetes cluster.

    kubectl create -f 2-elasticsearch_statefulset.yaml

Monitor the StatefulSet as it is rolled out.

    kubectl rollout status sts/es-cluster --namespace=kube-logging

Once all the Pods have been deployed, you can check that your Elasticsearch cluster is functioning correctly by performing a request against the REST API.

    kubectl port-forward es-cluster-0 9200:9200 --namespace=kube-logging

Then, in a separate terminal window, perform a curl request against the REST API:

    curl http://localhost:9200/_cluster/state?pretty

# Step 3 — Creating the Kibana Deployment and Service

    kubectl create -f 3-kibana-deployment.yaml

You can check that the rollout succeeded by running the following command:

    kubectl rollout status deployment/kibana --namespace=kube-logging

To access the Kibana interface, we'll once again forward a local port to the Kubernetes node running Kibana. Grab the Kibana Pod details using kubectl get:

    kubectl get pods --namespace=kube-logging

Forward the local port 5601 to port 5601 on this Pod:

    kubectl port-forward kibana-6c9fb4b5b7-plbg2 5601:5601 --namespace=kube-logging

Now, in your web browser, visit the following URL:

    http://localhost:5601

# Step 4 — Creating the Fluentd DaemonSet

Set up Fluentd as a DaemonSet, which is a Kubernetes workload type that runs a copy of a given Pod on each Node in the Kubernetes cluster. Using this DaemonSet controller, we'll roll out a Fluentd logging agent Pod on every node in our cluster.

In Kubernetes, containerized applications that log to stdout and stderr have their log streams captured and redirected to JSON files on the nodes. The Fluentd Pod will tail these log files, filter log events, transform the log data, and ship it off to the Elasticsearch logging backend we deployed in Step 2.

In addition to container logs, the Fluentd agent will tail Kubernetes system component logs like kubelet, kube-proxy, and Docker logs. 

To see a full list of sources tailed by the Fluentd logging agent, consult the kubernetes.conf file used to configure the logging agent.

    https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/docker-image/v0.12/debian-elasticsearch/conf/kubernetes.conf

We use the Fluentd DaemonSet spec provided by the Fluentd maintainers:

    https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/fluentd-daemonset-elasticsearch-rbac.yaml

Another helpful resource provided by the Fluentd maintainers is Kubernetes Logging with Fluentd:

    https://docs.fluentd.org/v0.12/articles/kubernetes-fluentd

## Create RBAC:

    kubectl create -f 4-fluentd-rbac.yaml

## Create a DaemonSet.
Here, we match the app: fluentd label defined in .metadata.labels and then assign the DaemonSet the fluentd Service Account. We also select the app: fluentd as the Pods managed by this DaemonSet.

Next, we define a NoSchedule toleration to match the equivalent taint on Kubernetes master nodes. This will ensure that the DaemonSet also gets rolled out to the Kubernetes masters. If you don't want to run a Fluentd Pod on your master nodes, remove this toleration.

Next, we configure Fluentd using some environment variables:

* FLUENT_ELASTICSEARCH_HOST: We set this to the Elasticsearch headless Service address defined earlier: elasticsearch.kube-logging.svc.cluster.local. This will resolve to a list of IP addresses for the 3 Elasticsearch Pods. The actual Elasticsearch host will most likely be the first IP address returned in this list. To distribute logs across the cluster, you will need to modify the configuration for Fluentd’s Elasticsearch Output plugin. To learn more about this plugin, consult Elasticsearch Output Plugin.
* FLUENT_ELASTICSEARCH_PORT: We set this to the Elasticsearch port we configured earlier, 9200.
* FLUENT_ELASTICSEARCH_SCHEME: We set this to http.
* FLUENT_UID: We set this to 0 (superuser) so that Fluentd can access the files in /var/log.

specify a 512 MiB memory limit on the FluentD Pod, and guarantee it 0.1vCPU and 200MiB of memory. You can tune these resource limits and requests depending on your anticipated log volume and available resources.

Next, we mount the /var/log and /var/lib/docker/containers host paths into the container using the varlog and varlibdockercontainers volumeMounts. These volumes are defined at the end of the block.

The final parameter we define in this block is terminationGracePeriodSeconds, which gives Fluentd 30 seconds to shut down gracefully upon receiving a SIGTERM signal. After 30 seconds, the containers are sent a SIGKILL signal. The default value for terminationGracePeriodSeconds is 30s, so in most cases this parameter can be omitted. 

    kubectl create -f 5-fluentd-daemonset.yaml

Verify that your DaemonSet rolled out successfully

    kubectl get ds --namespace=kube-logging

