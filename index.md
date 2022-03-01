# Revisioned Configmaps for Canary deployments on Kubernetes
Once we started deploying services on Kubernetes with Canary, we faced an issue during some of our rollouts.
We run most of our production nodes on Spot instances, we perform Canary deployments using Argo rollouts, our rollout strategy is very basic, we replace one pod, we perform manual tests on that pod and if the tests succeeded we continue to 100%.

## Problem:
- While performing the simplest canary deployment using Argo Rollouts we used 2 steps:
    ```
    steps:
    - setWeight: 1
    - pause: { }
    ```
- The Canary deployment is triggered by a configmap change.
- We use the [Automatic roll deployment](https://helm.sh/docs/howto/charts_tips_and_tricks/#automatically-roll-deployments) annotation on our pods:

    `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}`

    This annotation detect a change in the configmap and triggers a rolling restart of the pods.
- The pod from the new replicaSet boots and load the new configmap.
- There is a problem with the configuration and the new pod is either failing to start or there is a performance degradation, the service owner noticed that somethine is not right and start to investigate the pod's behavior.
- During the investigation, there are spot replacements in the cluster, and pods from the old replicaSet are being restarted.
- Those pods (From the old replicaSet), boots and load the new configmap with the faulty configuration and fail to start, causing downtime.

## The solution - revisioned configmaps:
To create and maintain the full lifecycle of revisioned configmaps, we needed to solve the folllowing issues:=
- Have a unique name of each configmap.
- Having unique name for each configmap that changes every deployment, triggered two other issues:
- Mounting the configmap to a pod - it was imposslbe to mount the configmaps by a static name.

    To solve it we implemented auto-mount mechanizem in our helm charts.
- Cleanup of old configmaps - We started to have leftovers of old configmaps.
To solve it we created a job that runs as part of every Canary deployment.
That Job gets the nameps of the revisioned configmaps that were created as part of this deployment and the job attach each configmap to the latest replicaset using ownerReferance.
Attaching the configmaps to the replicaSets, allowed us to utilized the same cleanup process Kubernetes perform for replicaSets on our configmaps.



You can use the [editor on GitHub](https://github.com/liorfranko/revisioned-configmaps.github.io/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [Basic writing and formatting syntax](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/liorfranko/revisioned-configmaps.github.io/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
