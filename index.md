# Revisioned Configmaps for Canary deployments on Kubernetes
We run most of our Kubernetes nodes on Spot instances.
We de our deployments using Argo rollouts, we perform a pretty simple Canary, where we replace one pod in the service and pause, once the new pod is up we perform manual tests and if the tests succeeded we continue to 100% rollout.
Once we started deploying services on Kubernetes with this Canary, we figured we also need to revision our configmaps.

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

    `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}`

## The problem
1. We start a deployment by changing something in a configmap.
2. The pod from the new replicaSet boots and load the new configmap.
3. The configmap is faulty and the new pod is either failing to start or there is a performance degradation.
4. The service owner noticed the issue and starts to investigate the pod's behavior.
5. During the investigation, there are spot replacements in the cluster, and pods from the old replicaSet are being restarted.

*While pods from the old replicaSet restarted, they load the new configmap with the faulty configuration.*

## The solution - revisioned configmaps:
To create and maintain the full lifecycle of revisioned configmaps, we needed to solve the following:
- Create a unique name of each configmap that changes every deployment.

    Having unique name for each configmap that changes every deployment, intreduced two issues:
- Mounting the revisioned configmap - it was imposslbe to mount the configmaps using a static name.

    To solve it we implemented auto-mount mechanizem in our helm chart.
- Cleanup of old configmaps - We started to have leftovers of old configmaps.
To solve it we created a job that runs as part of every Canary deployment.
That Job gets the names of the revisioned configmaps that were created as during the deployment, and attach each configmap to the latest replicaset using ownerReferance.
Attaching the configmaps to the replicaSets, allowed us to utilized the same cleanup process Kubernetes perform for replicaSets on our configmaps.



