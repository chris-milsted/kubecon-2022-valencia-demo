# Steps for Kubecon

This is a walkthrough on how to setup the demo and covers:
1. Creation and Installation of an EKS clusters using machines with local NVMe drives
2. Installation of the kubectl plugin and then installation of the Ondat CSI plugin
3. Installation of the workloads used in the demo, Cassandra using the K8ssandra plugin and Prometheus.
4. Clean up of the cluster.

End to end this should take about 1-2 hours including the failover testing.

# Step by step

## Setup AWS profile, intstall AWS command line, install eksctl and kubectl + plugin

I am using a Linux desktop, you should follow the instructions from the relevant webpages for the install the tooling onto your machine or jumpbox from where you will run the install, for reference the versions I used were:
```console
$ aws --version 
aws-cli/2.4.26 Python/3.8.8 Linux/5.17.4-200.fc35.x86_64 exe/x86_64.fedora.35 prompt/off
$ eksctl version
0.87.0
```
I also have a named profile I am using to assume roles in another AWS account for security which is specified in my `~/.aws/config` file, to pickup this setting with `eksctl` you can run the following in your terminal session ahead of the rest of the steps:
```bash
export AWS_PROFILE=<name>
```

I was using version 1.21 of Kubernetes with the EKS clusters, to make sure I did not have any version issues, I also installed 1.21 of the kubectl command as per the K8s webpages. Please make sure you have this version 

Last step is to install the kubectl storageOS plugin which Ondat uses for installation. You can find this here `https://github.com/storageos/kubectl-storageos` and there is a `curl` based installation command to install this onto your machine. For reference I used version `1.2.0` of the kubectl plugin which installs and is aligned to Ondat version `2.7.0`

## Notes about eksctl and the options used

To spin up the single region and single cloud provider demo, I am using the following eksctl manifest. Note that I am using the Ubuntu images as these contain the Linux I/O subsystem drivers. These are currently being evaluated for AL2022 and are needed for many storage platforms such as Ondat. You can follow the issue here [[Feature Request] - TCMU Userspace Kernel Module for Amazon EKS optimised AMI](https://github.com/amazonlinux/amazon-linux-2022/issues/88)

You can find the manifest here:
[Link to eksctl manifest](./eks-demo-chris.yaml)

You will see that I am also using the `PreBootstrapCommands` capability of eksctl with un-managed nodegroups to automatically locate the NVMe drive and format/mount it in a place that Ondat can then make use of it. There is an Open source project called [discoblocks](https://github.com/ondat/discoblocks) which is being developed to automate the management of block devices on any platform which will be used in the future.

There are probably a couple more steps you would want to take to make this a more robust solution in the current form if you intend to use this for more than a demo, for example adding the NVMe mount to the fstab to make it permanent across a reboot, using something like

```bash
sed -i[old]  'a /dev/nvme2n1       /var/lib/storageos         ext4      defaults  0 2 ' ./fstab
```
Happy to get any PR's to add this and any others to the these steps of course...

## Create you EKS cluster using i3en instances with local NVMe drives.

Run the following command:
```bash
eksctl create cluster --config-file eks-demo-chris.yaml
```
And then go and get a cup of coffee/tea/water as you wait for the cluster to spin up. Usually takes about 15-25 minutes so be make sure you do this ahead of when you need the cluster.
The command will also overwrite your `~.kube/config` file, you can also choose to have `eksctl` write to a different `kubeconfig` file if desired. See the `--kubeconfig` flag.

## Install Ondat into your EKS cluster
We should not have some EKS cluster nodes which have the NVMe mounted ready for Ondat to make use of. To generate the install command and to connect the cluster to the Ondat portal so that I can generate a trial license key new users will need to create an account at the [Ondat Portal](https://portal.ondat.io) website. You can do this with email or also using google or github credentials. 

Once you have created an account navigate to the dashboard page here `https://portal.ondat.io/dashboard` and click on the `Add Cluster` button. This will generate the `kubectl-storageos` plugin command which you should copy. The Command should look similar to the one below, I have added a couple more options just to specify which storage type to use for the self-hosted etcd cluster which Ondat uses and also to specify the admin username and password.

```bash
kubectl storageos install  \
    --include-etcd \
    --etcd-namespace storageos \
    --etcd-storage-class gp2 \
    --admin-username storageos \
    --admin-password storageos \
    --enable-portal-manager=true \
    --portal-client-id=3720861b-1234-45ab-82b0-bc5dbd6a011d \
    --portal-secret=b95daad1-1234-4ef2-9804-aaef767803577 \
    --portal-api-url=https://portal-setup-7dy4nlkxbq-ew.a.run.app \
    --portal-tenant-id=chris
```
Assuming your kubeconfig points to the EKS cluster, then you can simply run the command and this should complete the install of Ondat which you can validate by comparing the output of the command below. This step usuall takes about 30-60 seconds I have found.
```bash
$ kubectl get pods -n storageos
NAME                                                 READY   STATUS    RESTARTS   AGE
storageos-api-manager-54df545cbf-26jvv               1/1     Running   0          22s
storageos-api-manager-54df545cbf-p6h8g               1/1     Running   1          22s
storageos-csi-helper-65db657d7c-rwk87                3/3     Running   0          22s
storageos-etcd-0-zbnvz                               1/1     Running   0          104s
storageos-etcd-1-bwzcp                               1/1     Running   0          104s
storageos-etcd-2-jt7g7                               1/1     Running   0          104s
storageos-etcd-controller-manager-687d849ff6-tz72h   1/1     Running   0          2m
storageos-etcd-proxy-55479b544c-m45jx                1/1     Running   0          2m1s
storageos-node-djpjx                                 3/3     Running   0          81s
storageos-node-klq8l                                 3/3     Running   0          81s
storageos-node-qqn7w                                 3/3     Running   0          81s
storageos-operator-6fc687d97b-p9k57                  2/2     Running   0          2m
storageos-scheduler-f95895c6b-zwx6r                  1/1     Running   0          90s
```
Once you have all of the pods running we are ready to start using the K8s cluster.

## Create storage class for Promethues which is replicated and AZ aware

Use the following command below to define another storage class which we have added a couple of labels to the storage class to tell Ondat what features to enable on the storage class. You can find these all here (https://docs.ondat.io/docs/reference/labels/), in this scenario we are going to enable replication (so as well as the master have two replicas of data) and topology awareness which will make sure the master and replicas are in different Availability Zones.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storageos-rep-az
provisioner: csi.storageos.com
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/fstype: ext4
  storageos.com/replicas: "2"
  storageos.com/topology-aware: "true"
  csi.storage.k8s.io/secret-name: storageos-api
  csi.storage.k8s.io/secret-namespace: storageos
EOF
```

## Workloads on our cluster

Now that we have installed the cluster, Ondat and setup the storgae classes, we are ready to start with our application workloads. We have a couple of applications which are Prometheus and Cassandra. Lets start with Cassandra.

We are using the open source K8ssandra operator, [k8ssandra/k8ssandra-operator](https://github.com/k8ssandra/k8ssandra-operator), you can find the documentation on how to use this here [k8ssandra docs](https://docs-v2.k8ssandra.io/install/local/single-cluster-kustomize/). 

### Steps to install 

The documentation at the time of building the demo specified that certificate manager was needed before installing the k8ssandra operator. I did this using `kubectl apply` based on the documentation on the jetstack repository. Run the following command to do this (note the version I used):

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.2/cert-manager.yaml
```

Keep an eye on the cert-manager namespace and once the pods are all happy and running, you can then install the k8ssandra operator using a `kustomise` command with a `kubectl apply` as below:

```bash
kustomize build "github.com/k8ssandra/k8ssandra-operator/config/deployments/control-plane?ref=v1.0.0" | kubectl apply --server-side -f -
```

This will install the k8ssandra operator and while this finishes setting up, we will flip to installing prometheus and then come back to this and create some `K8ssandraCluster` objects that the operator will allow us to define.

## Install prometheus

We are going to install prometheus and from this we should start to see metrics from our cluster. In the demo we were using the `Lens` IDE so that we can visualise the cluster and perform actions like cordon and drain of nodes with simple button presses.

## add prometheus Helm repo
The first step would be to add the helm charts to your jump box and the also install the `helm` binary if you do not have this. I already had `helm` available so I just needed to add the community charts which you can do with the following command:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
This will add the prometheus charts to you local helm repository and then, as we have our Kubeconfig pointing to the EKS cluster, we can just tun a simple `helm install`. One point to note, we want to make sure prometheus is using our replicated AZ aware storage class that we created with Ondat. This will allow us to cordon and drain nodes or even kill servers and still make sure we have copies of the data in another AZ. We are using Ondat to turn Ephemeral NVMe drives into super fast 3AZ replicated fault tolerant storage. To install prometheus with the storgae class set run the following:

```bash
kubectl create namespace prometheus

helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="storageos-rep-az" \
    --set server.persistentVolume.storageClass="storageos-rep-az" 
```

## Install a Cassandra cluster

I now want to create a Cassandra cluster using the K8ssandraCluster CRD through the operator. To do this I ran the following, note that I am also installing a stargate POD which we will use for the endpoint of our nosqlbench workload.

```yaml
cat <<EOF | kubectl -n k8ssandra-operator apply -f -
apiVersion: k8ssandra.io/v1alpha1
kind: K8ssandraCluster
metadata:
  name: test
spec:
  cassandra:
    serverVersion: 4.0.3
    datacenters:
      - metadata:
          name: dc1
        size: 3
        podTemplateSpec:
          metadata:
            labels:
              storageos.com/fenced: "true"
        stargate:
          size: 1 
        storageConfig:
          cassandraDataVolumeClaimSpec:
            storageClassName: storageos
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 100Gi
        racks:
        - name: rack1
          nodeAffinityLabels:
            topology.kubernetes.io/zone: eu-west-1a
        - name: rack2
          nodeAffinityLabels:
            topology.kubernetes.io/zone: eu-west-1b
        - name: rack3
          nodeAffinityLabels:
            topology.kubernetes.io/zone: eu-west-1c
        config:
          jvmOptions:
            heapSize: 16Gi
EOF
```
# Checkpoint

At this point you should have.....
* An EKS cluster running with 6 worker nodes which are I3en based.
* Ondat installed and six healthy node pods in the storageos namespace.
* Prometheus installed and running in the prometheus namespace with cluster metrics appearing.
* The K8ssandra operator deployed and running and now also a Cassandra cluster running in the k8ssandra-operator namespace.

If this is the case we can now move onto starting a workload against Cassandra and then break things.

## Ondat Cli

First, I am going to deploy the Ondat CLI as well to allow me to dump low level information such as where the master and replica volumes are located. This will allow me to make sure I kill the nodes for maximum effect, i.e. kill a node with a master volume and workload. This way I know we will have to fail over the storage to a replica.

```yaml
cat <<EOF | kubectl -n storageos apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cli
  namespace: storageos
  labels:
    app: storageos-cli
    kind: storageos
    app.kubernetes.io/part-of: storageos
    app.kubernetes.io/component: storageos-cli
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storageos-cli
  template:
    metadata:
      name: storageos-cli
      labels:
        app: storageos-cli
    spec:
      containers:
      - image: storageos/cli:v2.7.0
        command:
          - "/bin/sh"
          - "-c"
          - "while true; do sleep 3600; done"
        imagePullPolicy: IfNotPresent
        name: cli
        ports:
        - containerPort: 5705
        env:
        - name: STORAGEOS_USERNAME
          value: storageos
        - name: STORAGEOS_PASSWORD
          value: storageos
        - name: STORAGEOS_ENDPOINTS
          value: storageos:5705
      restartPolicy: Always
EOF
```
if you want to dump out the details volume information the following commands will do this for you:
```bash
kubectl exec -ti $(kubectl get pods -n storageos -l app=storageos-cli -o=jsonpath='{.items[0].metadata.name}') -n storageos -- storageos describe volume -n prometheus

kubectl exec -ti $(kubectl get pods -n storageos -l app=storageos-cli -o=jsonpath='{.items[0].metadata.name}') -n storageos -- storageos describe volume -n k8ssandra-operator
```

## Cassandra cluster superuser password for nosqlbench to use
First step for the workload generator is to retrieve the superuser password for our Cassandra cluster. We called this test in the Custom Resource we specified above, so the username for this database storage as a secret will be "test-superuser". To get this we can run:

```bash
PASSWORD=$(kubectl get secrets -n k8ssandra-operator test-superuser --template={{.data.password}} | base64 --decode)
```
You can see the password by running `echo $PASSWORD`. I set this as a variable as the next script will use this password.

## SQL Bench job spec

Before you start this benchmark as it will run for about 25 minutes, make sure you are ready to commence your testing. I also opened up another terminal so that I could run kubectl commands, the lens IDE and my AWS console. Once you have all these ready we can start killing workloads.

To launch the nosqlbench I used the following, NOTE that it has been detuned massively, i.e. I am not using this to benchmark my cluster, rather it is just applying a consistent background workload to the cluster so that we can prove Cassandra is still responding to client queries even if we nuke nodes in the K8s cluster on which it is running:

```YAML
cat <<EOF | kubectl -n k8ssandra-operator apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: nosqlbench-throughput
  namespace: k8ssandra-operator
spec:
  template:
    spec:
      containers:
      - command:
        - java
        - -jar
        - nb.jar
        - cql-tabular2
        - username=test-superuser
        - password=$PASSWORD
        - rampup-cycles=10k
        - main-cycles=10M
        - write_ratio=5
        - read_ratio=5
        - async=5
        - hosts=test-dc1-stargate-service
        - --progress
        - console:1s
        - --report-csv-to
        - /var/lib/stress/throughput_test_io
        - rf=3
        - partsize=5000
        - -v
        - write_cl=LOCAL_ONE
        - read_cl=LOCAL_ONE
        image: nosqlbench/nosqlbench
        name: nosqlbench
        resources:
          requests:
            cpu: "2000m"
            memory: "4Gi"
        volumeMounts:
        - name: stress-results
          mountPath: /var/lib/stress
      volumes:
      - name: stress-results
        hostPath:
          path: /tmp
          type: DirectoryOrCreate
      restartPolicy: Never
EOF
```
Once the workloads are running I used the `kubectl logs` with the `-f` flag to follow the nosqlbench workload running. You can also delete the workload and re-deploy it if you run out time for this. The follow and delete commands are below and then you can re-deploy as above.
```bash
kubectl logs -n k8ssandra-operator jobs/nosqlbench-throughput -f
kubectl delete jobs -n k8ssandra-operator nosqlbench-throughput
```

## Node killing and speeding the process up
One of the tests in the demo is to kill a whole k8s node in the AWS console. In normal operations this will trigger a 300 second timeout before K8s will react to allow for transient network errors and partitions. As part of the Ondat CSI platform, there is a fencing agent which can, when it detects a node removal, send a force kill to the pod which will speed up the failover and allow the volumes to be detached and reattached to the new pod. This is very useful to speed up failover, this can be achieved by adding the annotation `storageos.com/fenced=true` to the pods. The following three commands will do this for each of the pods in the array:

 ```bash
kubectl label pod $(kubectl get pods -n k8ssandra-operator -l app.kubernetes.io/instance=cassandra-test -o=jsonpath='{.items[0].metadata.name}') -n k8ssandra-operator "storageos.com/fenced=true"
kubectl label pod $(kubectl get pods -n k8ssandra-operator -l app.kubernetes.io/instance=cassandra-test -o=jsonpath='{.items[1].metadata.name}') -n k8ssandra-operator "storageos.com/fenced=true"
kubectl label pod $(kubectl get pods -n k8ssandra-operator -l app.kubernetes.io/instance=cassandra-test -o=jsonpath='{.items[2].metadata.name}') -n k8ssandra-operator "storageos.com/fenced=true"
```
## Destructive tests

At this point you can start doing your testing. I wanted to show a couple of scenarios where the CSI driver and disk replication was providing resilience and then another where the application was doing the same. The two failure scenarios were graceful using node cordon and drain... and then AWS console pull the plug out of the back of the machine. From the kubecon video you can see that I kept my workloads up (both traditional DB in prometheus and advanced cloud native DB in Cassandra) through both. All the data was on Ephemeral disks so what was shown in AWS would also work on any other cloud provider, on premise or any edge deployment.

## Clean up

Once you have finished you can clean up as below. Note that the eksctl script timeouts are not long enough to tear down all the workloads so it can fail sometimes leaving you to manually clean up the AWS objects. I recommend you remove the workloads as well as per the below.

```bash
kubectl -n k8ssandra-operator delete K8ssandraCluster/test
kubectl delete ns prometheus cert-manager k8ssandra-operator
kubectl storageos uninstall --include-etcd --etcd-namespace storageos
kustomize build "github.com/k8ssandra/k8ssandra-operator/config/deployments/control-plane?ref=v1.0.0" | kubectl delete  -f -
eksctl delete cluster --config-file eks-demo-chris.yaml 
```

## Testing bandwith for NVMe drives
As part of some additional testing, I wanted to check the bandwidth of these machines, there is a lot of variability in some cloud deployments I found from noisy neighbour workloads and what seemed to be network contention. For the 3-way replicated volume backing the prometheus pod I did a simple `dd` just to check that we could get good performance before starting the tets:
```bash
kubectl exec -ti prometheus-server-75848577fc-p9r6v -n prometheus -c prometheus-server -- sh

$ dd if=/dev/zero of=/data/test bs=1M count=1000 oflag=direct
1000+0 records in
1000+0 records out
1048576000 bytes (1000.0MB) copied, 2.473930 seconds, 404.2MB/s
```
To check the volumes and make sure they were in the three AZ's in the AWS Dublin datacentres we also dumped the Ondat describe volume information which you can see below:
```bash
Master:           
  ID                7da09892-24ca-4e28-8576-9b251f58bdd3                                               
  Node              ip-192-168-63-17.eu-west-1.compute.internal (f6900e16-fa94-4d0a-95fd-c7ea2c846e46) 
  Health            online                                                                             
  Topology Domain   eu-west-1a                                                                         
                                                                                                       
Replicas:         
  ID                0b80921e-9e21-4f0e-92be-3b91c0339645                                               
  Node              ip-192-168-12-235.eu-west-1.compute.internal (4c65d8cc-239b-42a6-9734-1d28b1ea66b5)
  Health            ready                                                                              
  Promotable        true                                                                               
  Topology Domain   eu-west-1c                                                                         
                                                                                                       
  ID                9c56a45a-f3f1-4310-b5cf-efdb1d6842a7                                               
  Node              ip-192-168-82-203.eu-west-1.compute.internal (2b233e4d-ef94-4383-9f32-e349b72709f6)
  Health            ready                                                                              
  Promotable        true                                                                               
  Topology Domain   eu-west-1b 
```
