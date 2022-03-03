# Revisioned Configmaps for Canary deployments on Kubernetes
We run most of our Kubernetes nodes on Spot instances, and we perform our deployments using Argo rollouts.

Our rollout strategy is a pretty simple Canary deployments, where we replace one pod in the service and pause, once the new pod is up we perform manual tests and if the tests succeeded we continue to 100% rollout.

Once we started deploying services on Kubernetes with this Canary strategy, we figured we also need to revision our configmaps during Canary deployments.

## Infrastructure overview:
1. We maintain our in-house Helm chart.
2. We use the following Argo Rollouts Canary strategy:
```
strategy:
    canary:
        maxSurge: 0
        maxUnavailable: 1
        steps:
        - setWeight: 1
        - pause: {}
```
3. To trigger restarts of pods when configmaps are being changed we use the [Automatic roll deployment](https://helm.sh/docs/howto/charts_tips_and_tricks/#automatically-roll-deployments) annotation on our Rollout:

```yaml
kind: Rollout
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```
## The behaviour without revisioned configmap
1. We start a canary deployment by changing the configmap.
2. The pod from the new replicaSet boots and load the new configmap.
3. The configmap is faulty and the new pod is either failing to start or there is a performance degradation.
4. The service owner noticed the issue and starts to investigate.
5. During the investigation, there are spot replacements in the cluster, and pods from the old replicaSet are being restarted.

## The problem
**When pods from the old replicaSet are restarted, the new configmap is being loaded ,and if the configmap is faulty many pods can have issues**

## The solution - revisioned configmaps:
This is how we created a full lifecycle of revisioned configmaps:
- Create a unique name of each configmap that changes every deployment.

    Having a configmap name that changes every deployment, intreduced two issues:
    - Mounting the revisioned configmap - it was imposslbe to mount the configmaps using a static name as the name is generated during helm redering.

        To solve it we implemented auto-mount mechanizem in our helm chart.
    - Cleanup of old configmaps - Having different name every deployment leavs leftovers of old configmaps.

      To solve it we created a job that runs as part of every Canary deployment.
      That Job gets the names of the revisioned configmaps that were created during the deployment.

      That job attach each configmap to the deployment's replicaSet using ownerReferance.

      **Attaching the configmaps to the replicaSets, allowed us to utilized the same cleanup process Kubernetes perform for replicaSets on our configmaps.**



## Example:
The following example is done with a simple version of our [Helm chart](https://github.com/liorfranko/base-app)
1. We'll start by deploying our Helm chart:
![](./start.png)

The chart creates:
- A Rollout object, with replicaSet `demo-76f5954475` and 3 pods.
- A configmap `demo-cm-2648648036`
- A configmap-attacher-job `configmap-attacher-job-demo-20220302214309`

We'll start by viewing the ownerReferences of one of the pods:
```
kubectl -n devops-apps-01 get pods demo-76f5954475-cqfh5 -o json | jq '.metadata.ownerReferences'
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "ReplicaSet",
    "name": "demo-76f5954475",
    "uid": "90083c29-b58c-4c25-b097-0f170ef59ba1"
  }
]
```
We can see that the configmap has the same ownerReferences to the replicaSet, this was done by the configmap-attacher-job.
```
kubectl -n devops-apps-01 get cm demo-cm-2648648036 -o json | jq '.metadata.ownerReferences'
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion: true,
    "controller: true,
    "kind: "ReplicaSet",
    "name: "demo-76f5954475",
    "uid: "90083c29-b58c-4c25-b097-0f170ef59ba1"
  }
]
```
Let's take a look at the whole configmap:
```
kubectl -n devops-apps-01 get cm demo-cm-2648648036 -o yaml
apiVersion: v1
data:
  configuration.json: |2-

    {
      "general": {
        "projectName": "demo service",
      }
    }
  key-1: value-1
kind: ConfigMap
metadata:
  creationTimestamp: "2022-03-02T21:43:09Z"
  labels:
    app: demo
    argocd.argoproj.io/instance: demo
  name: demo-cm-2648648036
  namespace: devops-apps-01
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: demo-76f5954475
    uid: 90083c29-b58c-4c25-b097-0f170ef59ba1
  resourceVersion: "323126811"
  selfLink: /api/v1/namespaces/devops-apps-01/configmaps/demo-cm-2648648036
  uid: 71b7f271-5e44-443b-959f-27c1d0f2d6b7
```
We can also see the auto-mounting of the configmap on the pod:
```
kubectl -n devops-apps-01 get pods demo-76f5954475-cqfh5 -o json | jq '.spec.volumes[0]'
{
  "name": "configmaps-volume",
  "projected": {
    "defaultMode": 420,
    "sources": [
      {
        "configMap": {
          "name": "demo-cm-2648648036"
        }
      }
    ]
  }
}
kubectl -n devops-apps-01 get pods demo-76f5954475-cqfh5 -o json | jq '.spec.containers[0].volumeMounts[0]'
{
  "mountPath": "/etc/kubernetes/configmaps",
  "name": "configmaps-volume"
}
kubectl -n devops-apps-01 exec -it demo-76f5954475-cqfh5 bash -- ls -l /etc/kubernetes/configmaps
total 0
lrwxrwxrwx 1 root root 25 Mar  2 21:43 configuration.json -> ..data/configuration.json
lrwxrwxrwx 1 root root 12 Mar  2 21:43 key-1 -> ..data/key-1
 ~  kubectl -n devops-apps-01 exec -it demo-76f5954475-cqfh5 bash -- cat /etc/kubernetes/configmaps/configuration.json

{
  "general": {
    "projectName": "demo service",
  }
}
```

2. Now let's perform a change in the configmap.
![](./step-2.png)
- We can see that a new replicaSet is created - `demo-58f56d4967`
- We can see that a new configmaps is created - `demo-cm-557795215`
- We can see that the configmap from the first deploy `demo-cm-2648648036`, is now shown below the old replicaSet `demo-76f5954475`
- We can see a new version of the configmap-attacher-job - `configmap-attacher-job-demo-20220302220158`

We can see that the new configmap is attached to the new replicaSet
```
kubectl -n devops-apps-01 get pods demo-58f56d4967-8kcwp -o json | jq '.metadata.ownerReferences'
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "ReplicaSet",
    "name": "demo-58f56d4967",
    "uid": "76a7970c-862d-4ff5-a387-cc41a367308c"
  }
]
```
The most important part at this point is the mounting of the configmaps, let's compare the mounting of an old pod and a new pod:
```
kubectl -n devops-apps-01 get pods demo-76f5954475-frwcb -o json | jq '.spec.volumes[0]'
{
  "name": "configmaps-volume",
  "projected": {
    "defaultMode": 420,
    "sources": [
      {
        "configMap": {
          "name": "demo-cm-2648648036"
        }
      }
    ]
  }
}
kubectl -n devops-apps-01 get pods demo-58f56d4967-8kcwp -o json | jq '.spec.volumes[0]'
{
  "name": "configmaps-volume",
  "projected": {
    "defaultMode": 420,
    "sources": [
      {
        "configMap": {
          "name": "demo-cm-557795215"
        }
      }
    ]
  }
}
```
Let's simulate a spot replacement by deleting one of the old pods
```
kubectl -n devops-apps-01 delete pods demo-76f5954475-frwcb
pod "demo-76f5954475-frwcb" deleted
```
![](./step-3.png)
```
kubectl -n devops-apps-01 get pods demo-76f5954475-6sdbb -o json | jq '.spec.volumes[0]'
{
  "name": "configmaps-volume",
  "projected": {
    "defaultMode": 420,
    "sources": [
      {
        "configMap": {
          "name": "demo-cm-2648648036"
        }
      }
    ]
  }
}
```
We can see that even thought we deployed new configmap in the cluster, old pods that are restarted use the old configmaps.

3. Let's promote the Rollout.

    All pods were moved from the old replicaSet to the new replicaSet
    ![](./step-4.png)
4. Let's create another change in the configmap.
    ![](./step-5.png)
    We can see another revioned configmap, another replicaSet, another configmap-attacher-job.

    We can see that the configmap from the old deployment were moved below the old replicaSet.
5. Let's promote the Rollout.

    All pods were moved from the old replicaSet to the new replicaSet
    ![](./step-6.png)
6. Let's simulate Kubernete GC, by deleting the old replicaSets `demo-76f5954475` and `demo-58f56d4967`.
```
kubectl -n devops-apps-01 get cm | grep demo
demo-cm-183520271                         4      5m5s
demo-cm-2648648036                        2      42m
demo-cm-557795215                         3      23m
kubectl -n devops-apps-01 delete replicasets.apps demo-76f5954475 demo-58f56d4967
replicaset.apps "demo-76f5954475" deleted
replicaset.apps "demo-58f56d4967" deleted
kubectl -n devops-apps-01 get cm | grep demo
demo-cm-183520271                         4      5m31s
```
![](./step-7.png)
We can see that old configmaps were successfully deleted!
