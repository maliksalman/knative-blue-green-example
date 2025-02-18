# App deployments with gradual traffic switchover using Knative 

## Initial Deployment

First deploy the initial revision of the `hello` service, it will be tagged as `blue` and 100% traffic will be routed to it:

In the manifest, the traffic section is defined as:

```yaml
  traffic:
    - latestRevision: true
      percent: 100
      tag: blue
```

Then apply the YAML:

```
kubectl apply -f hello.yml
```

## Subsequent deployments

Find out the latest revision name, make changes to the manifest and deploy. This will deploy a new revision called `green`, but 100% of the traffic will still be routed to the `blue` revision. Lets assume the current revision is called `hello-00001`, the traffic section in the manifest will be defined as:

```yaml
  traffic:
    - latestRevision: false
      revisionName: hello-00001
      percent: 100
      tag: blue
    - latestRevision: true
      percent: 0
      tag: green
```

Then apply the YAML:

```
kubectl apply -f hello.yml
```

## Happy path scenario

In this scenario, traffic will be transitioned to the new revision slowly and we will not need to rollback. To accomplish this, we slowly transition the traffic from `blue` to `green` revisions using the following convinient `kn` commands:

```
kn service update hello --traffic green=20  --traffic blue=80
kn service update hello --traffic green=40  --traffic blue=60
kn service update hello --traffic green=60  --traffic blue=40
kn service update hello --traffic green=80  --traffic blue=20
kn service update hello --traffic green=100  --traffic blue=0
```

By this time the `green` service will be receiving 100% of the traffic. Now we can remove the existing `blue` and `green` tags, and tag the most recent revision as `blue`. This is done so that we can repeat this process for deploying the next change:

```
kn service update hello --untag green,blue --tag @latest=blue
```

## Non-happy path scenario

In this scenario, traffic will be transitioned to the new revision slowly, but during that time we will discover that something is wrong with the new `green` revision. In that case, we will:

1. First send 100% of the traffic to the existing `blue` revision:

```
kn service update hello --traffic green=0  --traffic blue=100
```

2. Then we will untag the `green` revision:
```
kn service update hello --untag green
```

## Cleanup the un-referenced revisions

Multiple revisions of the same service are active at a time along with their resources (pods, etc). To reclaim resources for revisions that are not receiving any traffic and are untagged, we can issue the following command:

```
kn revisions delete --prune hello
```