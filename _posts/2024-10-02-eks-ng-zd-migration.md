---
title: AWS EKS zero-downtime Node Group migration
author: kirka
date: 2024-10-02 21:10:00 +0800
categories: [Software, Tutorial]
tags: [cloud, eks]
pin: true
render_with_liquid: false
---

Once I worked on service that allow customers to deploy their profiles (configuration for company healthcare server) in the cloud. \
Clients created environment and will be able to configure number `instances`, their `RAM`, `CPU` and so on.\
In backend actually every environment was a `EKS` cluster with `ingress`, `node/pod autoscaler`, `calico` and other fancy things.\
And as clients may rescale their environment at runtime (and of course they don't want to know what is behind that), we should be able to resize cluster flawlessly.\
Therefore, I had a task to automate migration all deployments to different `Node Group` without downtime and packet drops.

> Service was in `Java/Spring`, so `Java AWS SDK V2` was used (but it doesn't matter).
{: .prompt-tip }

## Prerequisites

We find-out that most of the clients used only 1–2 instances (`pods`) for their workload. \
Probably not what for kubernetes was created? \
And with that small number while updating/migrating pods, we very easily caught packet drops. \
Why? \
We from our service can't control what actually was deployed. We can't tell deployed profile `close your connection`. We still can stop profile (and pod) of course. \
And we can't control `Load Balancer` either (we used `Network Load Balancer`). \
So the main strategy for "No packed drops" for deployment configuration is big `termination grace period`. Bigger than any open connection can last for a specific pod.

## Steps

Let's go step by step how to achieve this.

### 1. Create Node Group

At first, we need to create new `Node Group` - the one we will migrate to:

```java
 LaunchTemplate launchTemplate = ec2Service.createLaunchTemplate(env, nodeGroupname, tags);
        if (launchTemplate == null) {
            log.error(String.format("error creating launchTemplate for env %s", env.getName()));
            return null;
        }

        var desiredSize = Math.min(env.getDesiredWorkerCount(), env.getMaxWorkerCount());

        var request = CreateNodegroupRequest
                .builder()
                .launchTemplate(builder -> builder.id(launchTemplate.launchTemplateId()))
                .version(cluster.getVersion())
                .clusterName(cluster.getName())
                .nodegroupName(nodeGroupname)
                .scalingConfig(builder -> builder.desiredSize(Math.max(desiredSize, env.getMinWorkerCount()))
                        .maxSize(env.getMaxWorkerCount())
                        .minSize(env.getMinWorkerCount()))
                .amiType(clusterConfig.getNodeAmi())
                .instanceTypes(List.of(clusterConfig.getWorkerInstanceType(env.getWorkerSize()).toString()))
                .subnets(clusterConfig.getNodeSubnets())
                .tags(tags)
                .labels(Map.of(WORKING_NODE, "true"))
                .nodeRole(clusterConfig.getNodeRole())
                .build();
        try {
            var nodegroup = eksClient.createNodegroup(request);
```
and code for Launch Template (`ec2Service.createLaunchTemplate`):
```java
            var request = CreateLaunchTemplateRequest
                    .builder()
                    .launchTemplateName(nodegroupName)
                    .tagSpecifications(tagSpecification)
                    .launchTemplateData(builder -> builder
                            .tagSpecifications(tagSpecificationRequest)
                            .blockDeviceMappings(LaunchTemplateBlockDeviceMappingRequest.builder()
                                    .deviceName("/dev/xvda")
                                    .ebs(ebsbuilder -> ebsbuilder
                                            .deleteOnTermination(true)
                                            .iops(3000)
                                            .throughput(150)
                                            .volumeType(VolumeType.GP3)
                                            .volumeSize(50)
                                            .encrypted(true)
                                    )
                                    .build()))
                    .build();

            return ec2.createLaunchTemplate(request).launchTemplate();
```

> At the time when I was worked on that, Java SDK has a bug (or feature?)—\
> It didn't allow creating a node-group without launching template.\
> You should have created an empty template even if you didn't need template customization.\
> Probably it was fixed already.
{: .prompt-warning }

For manual creation using CLI, you can refer [to AWS docs](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html).


### 2. Tagging the old Node Group

#### 2.1 No Schedule Taint

The next step is tagging your existing Node Group for stopping scheduling new pods on this group.\
Kubernetes has special taint (mark for nodes) for that - `No Schedule`.\
You can read about taints in kubernetes [docs](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).


#### 2.2 Froze Node Autoscaler

Another thing that you better do —froze your Node Autoscaler.\
It's not necessary but will eliminate any possibility of accidental node resizing.\
In time when your current node group is tainted but a new node group doesn't have enough nodes due to resizing\—you may find in a situation when next steps will be in an uncertain state.\
Simplest frozen algorithm—just set Node Autoscaler min and max node size property to the current nodes count.\
You can add annotation to Autoscaler to remember the previous state.

### 3. Scaling deployments

Now you should do actual migration.\
How to migrate deployments?\
Just scale them and due to old Node Group tainted as `No Schedule`, new pods will be created on new group.\
Simple as that.\
Almost.

#### 3.1 Tagging deployments

To simplify your life, remember to annotate deployment with their current state (number of pods). \
Will be handily on at the end of migration.

#### 3.2 Froze Pod Autoscaler

If you have Pod Autoscaler - do the same as you do with [Node Autoscaler](#22-froze-node-autoscaler).


#### 3.3 Scale service deployments

Next I would recommend scaling your service deployments(coreDns, Autoscaler),\
just to prevent race condition between deployments in rare case when your new node group doesn't have enough space.

#### 3.4 Scale user deployments

And finally, scale your actual deployments.
It's better to scale to x2 of initial size—just to have an exact copy of state after migration.

### 4. Tagging the old Node Group again

When deployments are scaled and ready (when ready!) proceed to exclusion old nodes from traffic. \
We need to exclude a node group from Load Balancer for clean old deployment from receiving traffic. \
Kubernetes has a label for that - `http://node.kubernetes.io/exclude-from-external-load-balancers`. Refer to [docs](https://kubernetes.io/docs/reference/labels-annotations-taints/) as usual.\

> Please note - if your LB service `externalTrafficPolicy` property set to `local` - you are fine.\
> But if your property is `cluster` - excluding from Load Balancer won't give you that effect (Still may be worth doing).
{: .prompt-warning }

 
### 5. Deleting the old Node Group

When you waited some time, and you are sure that nodes are excluded from Load Balancer - you can delete them.\
How to know depends on your LB configuration.
But if you know your Load Balancer `idle connection timeout` - wait at least few sec more than that value.
After that, drop Node Group.
\
EKS will evict Node Group and will respect `terminationGracePeriodSeconds` of your pods deployments.

### 6. Descaling Deployments

And right after delete Node Group - descale your Deployments.\
Often I saw when people recommend first descaling and delete Node Group after.\, 
But I encounter strange behavior which I didn't expected.\
When descaling by scale of 2 as example -
kubernetes (at least versions <= 1.27, but I'm pretty sure it's actual in 1.30)
may descale pods from old and new Node Group randomly.
Even if Node Group marked as `No Schedule`.\
Not good, approach "Delete and descale after" also risky—you would have better to do it fast.
Don't wait after deletion Node Group.\
Who knows what happens?\
When you delete first - EKS won't descale pods from the new Node Group.
It will drop already draining pods from the old one.\ 
Another drawback—it might schedule them in a new Node Group before you descale.\
So think what approach suits for you.

#### 6.1 Descale user deployments

Let's pretend that you do as I said above.\
Descale your deployments to the initial state.

#### 6.2 Descale service deployments

Descale your service deployments to the initial state.

#### 6.3 Untagging deployments

Untag deployments—remove annotations that you put in them in [previous steps](#31-tagging-deployments).

#### 6.4 Unfroze Pod Autoscaler

Restore state of Pod Autoscaler.

#### 6.5 Unfroze Node Autoscaler

And restore state of Node Autoscaler.

### Summary

It's all.\
I hope this helps someone.
Despite there a lot of tutorials in web.
