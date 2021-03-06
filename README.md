# K8s Third Party Resource and Operator For TensorFlow jobs

## Motivation

Distributed TensorFlow training jobs require managing multiple sets of TensorFlow replicas. 
Each set of replicas usually has a different role in the job. For example, one set acts
 as parameter servers, another provides workers and another provides a controller.
 
K8s makes it easy to configure and deploy each set of TF replicas. Various tools like
 [helm](https://github.com/kubernetes/helm) and [ksonnet](http://ksonnet.heptio.com/) can
 be used to simplify generating the configs for a TF job.
 
 However, in addition to generating the configs we need some custom control logic because
 K8s built-in controllers (Jobs, ReplicaSets, StatefulSets, etc...) don't provide the semantics
 needed for managing TF jobs.
 
 To solve this we define a 
 [K8S Third Party Resource](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-third-party-resource/)
 and [Operator](https://coreos.com/blog/introducing-operators.html) to manage a TensorFlow
 job on K8s.


TfJob provides a K8s resource representing a single, distributed, TensorFlow job. 
The Spec and Status (defined in [tf_job.go](https://cloud-ml.googlesource.com/jlewi-training/+/tpr/pkg/spec/tf_job.go))
are customized for TensorFlow. The spec allows specifying the Docker image and arguments to use for each TensorFlow
replica (i.e. master, worker, and parameter server). The status provides relevant information such as the number of
replicas in various states.

Using a TPR gives users the ability to create and manage TF Jobs just like builtin K8s resources. For example to
create a job

```
kubectl create -f examples/tf_job.yaml
```

To list jobs

```
kubectl get tfjobs

NAME          KINDS
example-job   TfJob.v1beta1.mlkube.io
```

## Design

The code is closely modeled on Coreos's [etcd-operator](https://github.com/coreos/etcd-operator).

The TfJob Spec(defined in [tf_job.go](https://cloud-ml.googlesource.com/jlewi-training/+/tpr/pkg/spec/tf_job.go)) 
reuses the existing Kubernetes structure PodTemplateSpec to describe TensorFlow processes. 
We use PodTemplateSpec because we want to make it easy for users to 
  configure the processes; for example setting resource requirements or adding volumes. 
  We expect
helm or ksonnet could be used to add syntactic sugar to create more convenient APIs for users not familiar
with Kubernetes.

Leader election allows a K8s deployment resource to be used to upgrade the operator.

## Installing the TPR and operator on your k8s cluster

1. Clone the repository

    ```
    git clone https://cloud-ml.googlesource.com/jlewi-training
    ```

1. Deploy the operator

   ```
   helm install tf-job-chart/ -n tf-job --wait --replace
   ```

1. Make sure the operator is running

    ```
    kubectl get pods
    
    NAME                               READY     STATUS    RESTARTS   AGE
    tf-job-operator-3083500267-wxj43   1/1       Running   0          48m

    ```

1. Run the helm tests

    ```
    helm test tf-job
    RUNNING: tf-job-tfjob-test-pqxkwk
    PASSED: tf-job-tfjob-test-pqxkwk
    ```
    
## Run the example

A simplistic TF program is in the directory tf_sample. 

1. Start the example

    ```
    helm install --name=tf-job ./examples/tf_job
    ```
    
1. Check the job

    ```
    kubectl get tfjobs -o yaml
    ```

## Project Status

This is very much a prototype.

### Logging

Logging still needs work.

We'd like to tag log entries with semantic information suitable for TensorFlow. For example, we'd like to tag entries with metadata indicating the
replica that produced the log entry. There are two issues here

1. Tagging Tensorflow entries with appropriate semantic information

    * Usinge Python sitecustomize.py might facilitate injecting a custom log handler that outputs json entries.
    * For parameter servers, we might want to just run the TensorFlow standard server and its not clear how we
      would convert those logs to json.
      
1. Integrate with Kubernetes cluster level logging.

    * We'd like the logs to integrate nicely with whatever cluster level logging users configure.
    * For example, on GCP we'd like the log entries to be automatically streamed to Stackdriver and indexed by the
      TensorFlow metadata to facilitate querying e.g. by replica.
    * GCP's fluentd logger is supposed to automatically handle JSON logs

Additionally, we'd like TensorFlow logs to be available via

```
kubectl logs
```

So that users don't need to depend on cluster level logging just to see basic logs.

In the current implementation, pods aren't deleted until the TfJob is deleted. This allows standard out/error to be fetched
via kubectl. Unfortunately, this leaves PODs in the RUNNING state when the TfJob is marked as done which is confusing. 

### Status information

The status information reported by the operator is hacky and not well thought out. In particular, we probably
need to figure out what the proper phases and conditions to report are.

### Failure/Termination Semantics

The semantics for aggregating status of individual replicas into overall TfJob status needs to be thought out.

### Dead/Unnecessary code

There is a lot of code from earlier versions (including the ETCD operator) that still needs to be cleaned up.

### Testing

There is minimal testing.

#### Unittests

There are some unittests.

#### E2E tests

The helm package provides some basic E2E tests.

### TensorBoard Integration

What's the best way to integrate TensorBoard?

  *  A TfJob could launch TensorBoard and the lifetime of the TensorBoard instance would be tied to the job.
  *  In addition we could possible use a TPR for TensorBoard or a framework like []fission.io](http://fission.io/) to
     make it easy to launch a TensorBoard instance just by specifying the arguments for TensorBoard.

## Building the Operator

To build the agent pull in client-go

    ```
    go get k8s.io/client-go/...
    ```

Create a symbolic link inside your GOPATH to the location you checked out the code

    ```
    ln -sf ${GIT_TRAINING}/src/mlkube.io ${GOPATH}/src/
    ```

  * GIT_TRAINING should be the location where you checked out https://cloud-ml.googlesource.com/training/
	
Build it

```
go install mlkube.io/cmd/tf_operator
```

## Runing the Operator Locally

Running the operator locally (as opposed to deploying it on a K8s cluster) is convenient for debugging/development.

We can configure the operator to run locally using the configuration available in your kubeconfig to communicate with 
a K8s cluster.

Set your environment
```
export USE_KUBE_CONFIG=$(echo ~/.kube/config)
export MY_POD_NAMESPACE=default
export MY_POD_NAME=my-pod
```

TODO(jlewi): Do we still need to set MY_POD_NAME? Why?

## Go version

On ubuntu the default go package appears to be gccgo-go which has problems see [issue](https://github.com/golang/go/issues/15429) golang-go package is also really old so install from golang tarballs instead.
