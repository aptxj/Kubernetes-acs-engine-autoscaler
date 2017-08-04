
[![CircleCI](https://circleci.com/gh/wbuchwalter/Kubernetes-acs-engine-autoscaler.svg?style=svg)](https://circleci.com/gh/wbuchwalter/Kubernetes-acs-engine-autoscaler)


:star2: This project is a fork of [OpenAI](https://openai.com/blog/)'s [Kubernetes-ec2-autoscaler](https://github.com/openai/kubernetes-ec2-autoscaler)  
 
:warning: **ACS is not supported, this autoscaler is for [`acs-engine`](https://github.com/azure/acs-engine) only**

:information_source: If you need autoscaling for VMSS, check out [OpenAI/kubernetes-ec2-autoscaler:azure](https://github.com/openai/kubernetes-ec2-autoscaler/tree/azure) or [cluster-autoscaler](https://github.com/kubernetes/contrib/tree/master/cluster-autoscaler)

# kubernetes-acs-engine-autoscaler

kubernetes-acs-engine-autoscaler is a node-level autoscaler for [Kubernetes](http://kubernetes.io/) for clusters created with acs-engine.  
Kubernetes is a container orchestration framework that schedules Docker containers on a cluster, and kubernetes-acs-autoscaler can scale based on the pending job queue.

## Getting started

Check out this blog post that will walk you through setting up the autoscaler: [Autoscaling a Kubernetes cluster created with acs-engine on Azure](https://medium.com/@wbuchwalter/autoscaling-a-kubernetes-cluster-created-with-acs-engine-on-azure-5e24ddc6402e)

## Architecture

![Architecture Diagram](docs/kubernetes-acs-autoscaler.png)

## Setup

The autoscaler can be run anywhere as long as it can access the Azure
and Kubernetes APIs, but the recommended way is to set it up as a
Kubernetes Pod.

### Credentials

You need to provide a Service Principal to the autoscaler.
You can create one using [Azure CLI](https://github.com/Azure/azure-cli):  
`az ad sp create-for-rbac`

You also need to provide the `kubeConfigPrivateKey`, `clientPrivateKey` and `caPrivateKey`. You can find these values in the `azuredeploy.parameters.json` that you generated by `acs-engine`.  

### Run in-cluster

The best way to provide the credentials in Kubernetes is with a [secret](http://kubernetes.io/docs/user-guide/secrets/).

Here is a sample format for `secret.yaml`:
```
apiVersion: v1
kind: Secret
metadata:
  name: autoscaler
data:
  azure-sp-app-id: [base64 encoded app id]
  azure-sp-secret: [base64 encoded secret access key]
  azure-sp-tenant-id: [base64 encoded tenand id]
  kubeconfig-private-key: [base64 encoded kubeConfigPrivateKey]
  client-private-key: [base64 encoded clientPrivateKey]
  ca-private-key: [base64 encoded caPrivateKey]
```

You can then save it in Kubernetes:
```
$ kubectl create -f secret.yaml
```

[scaling-deployment.yaml](scaling-deployment.yaml) has an example
[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
that will set up Kubernetes to always run exactly one copy of the autoscaler.
To create the Deployment:
```
$ kubectl create -f scaling-deployment.yaml
```
> NOTE: If you provided a custom deployment name when deploying the kubernetes cluster, then you can pass in the deployment name by adding the `--acs-deployment` flag in the command section of the yaml file. Otherwise, it will look for the default `azuredeploy` deployment.

You should then be able to inspect the pod's status and logs:
```
$ kubectl get pods -l app=autoscaler
NAME               READY     STATUS    RESTARTS   AGE
autoscaler-opnax   1/1       Running   0          3s

$ kubectl logs autoscaler-opnax 
2016-08-25 20:36:45,985 - autoscaler.cluster - DEBUG - Using kube service account
2016-08-25 20:36:45,987 - autoscaler.cluster - INFO - ++++++++++++++ Running Scaling Loop ++++++++++++++++
2016-08-25 20:37:04,221 - autoscaler.cluster - INFO - ++++++++++++++ Scaling Up Begins ++++++++++++++++
...
```

### Running locally
```
$ docker build -t autoscaler .
$ ./devenvh.sh
#in the container
$ python main.py --resource-group k8s --service-principal-app-id 'XXXXXXXXX' --service-principal-secret 'XXXXXXXXXXXXX' service-principal-tenant-id 'XXXXXX' -vvv --kubeconfig /root/.kube/config --kubeconfig-private-key 'XXXX' --client-private-key 'XXXX'
```


## Full List of Options

```
$ python main.py [options]
```
- --resource-group: Name of the resource group containing the cluster
- --kubeconfig: Path to kubeconfig YAML file. Leave blank if running in Kubernetes to use [service account](http://kubernetes.io/docs/user-guide/service-accounts/).
- --service-principal-app-id: Azure service principal id. Can also be specified in environment variable `AZURE_SP_APP_ID`
- --service-principal-secret: Azure service principal secret. Can also be specified in environment variable `AZURE_SP_SECRET`
- --service-principal-tenant-id: Azure service princiap tenant id. Can also be specified in environment variable `AZURE_SP_TENANT_ID`
- --kubeconfig-private-key: The key passed to the `kubeConfigPrivateKey` parameter in your `azuredeploy.parameters.json` generated with `acs-engine`
- --client-private-key: The key passed to the `clientPrivateKey` parameter in your `azuredeploy.parameters.json` generated with `acs-engine`
- --ca-private-key: The key passed to the `caPrivateKey` parameter in your `azuredeploy.parameters.json` generated with `acs-engine`
- --sleep: Time (in seconds) to sleep between scaling loops (to be careful not to run into AWS API limits)
- --slack-hook: Optional [Slack incoming webhook](https://api.slack.com/incoming-webhooks) for scaling notifications
- --dry-run: Flag for testing so resources aren't actually modified. Actions will instead be logged only.
- -v: Sets the verbosity. Specify multiple times for more log output, e.g. `-vvv`
- --debug: Do not catch errors. Explicitly crash instead.
- --ignore-pools: Names of the pools that the autoscaler should ignore, separated by a comma.
- --spare-agents: Number of agent per pool that should always stay up (default is 1)
- --acs-deployment: The name of the deployment used to deploy the kubernetes cluster initially
- --idle-threshold: Maximum duration (in seconds) an agent can stay idle before being deleted
- --util-threshold: Utilization of a node in percent under which it is considered under utilized and should be cordoned
- --over-provision: Number of extra agents to create when scaling up, default to 0.

## Windows Machine Pools

Currently node pools with Windows machines are not supported. If a Windows pool is part of the deployment the autoscaler will fail even for scaling Linux-based node pools.
