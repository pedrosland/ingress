# GCE Ingress controller FAQ

This page contains general FAQ for the gce Ingress controller.

Table of Contents
=================

* [How do I deploy an Ingress controller?](#how-do-i-deploy-an-ingress-controller)
* [I created an Ingress and nothing happens, now what?](#i-created-an-ingress-and-nothing-happens-now-what)
* [What are the cloud resources created for a single Ingress?](#what-are-the-cloud-resources-created-for-a-single-ingress)
* [The Ingress controller events complain about quota, how do I increase it?](#the-ingress-controller-events-complain-about-quota-how-do-i-increase-it)
* [Why does the Ingress need a different instance group then the GKE cluster?](#why-does-the-ingress-need-a-different-instance-group-then-the-gke-cluster)
* [Why does the cloud console show 0/N healthy instances?](#why-does-the-cloud-console-show-0n-healthy-instances)
* [Can I configure GCE health checks through the Ingress?](#can-i-configure-gce-health-checks-through-the-ingress)
* [Why does my Ingress have an ephemeral ip?](#why-does-my-ingress-have-an-ephemeral-ip)
* [Can I pre-allocate a static-ip?](#can-i-pre-allocate-a-static-ip)
* [Does updating a Kubernetes secrete update the GCE TLS certs?](#does-updating-a-kubernetes-secrete-update-the-gce-tls-certs)
* [Can I tune the loadbalancing algorithm?](#can-i-tune-the-loadbalancing-algorithm)
* [Is there a maximum number of Endpoints I can add to the Ingress?](#is-there-a-maximum-number-of-endpoints-i-can-add-to-the-ingress)
* [How do I match GCE resources to Kubernetes Services?](#how-do-i-match-gce-resources-to-kubernetes-services)
* [Can I change the cluster UID?](#can-i-change-the-cluster-uid)
* [Why do I need a default backend?](#why-do-i-need-a-default-backend)
* [How does Ingress work across 2 GCE clusters?](#how-does-ingress-work-across-2-gce-clusters)
* [I shutdown a cluster without deleting all Ingresses, how do I manually cleanup?](#i-shutdown-a-cluster-without-deleting-all-ingresses-how-do-i-manually-cleanup)
* [How do I disable the GCE Ingress controller?](#how-do-i-disable-the-gce-ingress-controller)


## How do I deploy an Ingress controller?

On GCP (either GCE or GKE), every Kubernetes cluster has an Ingress controller
running on the master, no deployment necessary. You can deploy a second,
different (i.e non-GCE) controller, like [this](README.md#how-do-i-deploy-an-ingress-controller).

## I created an Ingress and nothing happens, now what?

Please check the following:

1. Output of `kubectl describe`, as shown [here](README.md#i-created-an-ingress-and-nothing-happens-what-now)
2. Do your Services all have a `NodePort`?
3. Do your Services either serve a http 200 on `/`, or have a readiness probe
   as described in [this section](#can-i-configure-gce-health-checks-through-the-ingress)?
4. Do you have enough GCP quota?

## What are the cloud resources created for a single Ingress?

__Terminology:__

* [Global Forwarding Rule](https://cloud.google.com/compute/docs/load-balancing/http/global-forwarding-rules): Manages the Ingress VIP
* [TargetHttpProxy](https://cloud.google.com/compute/docs/load-balancing/http/target-proxies): Manages SSL certs and proxies between the VIP and backend
* [Url Map](https://cloud.google.com/compute/docs/load-balancing/http/url-map): Routing rules
* [Backend Service](https://cloud.google.com/compute/docs/load-balancing/http/backend-service): Bridges various Instance Groups on a given Service NodePort
* [Instance Group](https://cloud.google.com/compute/docs/instance-groups/): Collection of Kubernetes nodes

The pipeline is as follows:

```
Global Forwarding Rule -> TargetHTTPProxy
        |                                  \                               Instance Group (us-east1)
    Static IP                               URL Map - Backend Service(s) - Instance Group (us-central1)
        |                                  /                               ...
Global Forwarding Rule -> TargetHTTPSProxy
                            ssl cert
```

In addition to this pipeline:
* Each Backend Service requires a HTTP health check to the NodePort of the
  Service
* Each port on the Backend Service has a matching port on the Instance Group
* Each port on the Backend Service is exposed through a firewall-rule open
  to the GCE LB IP range (`130.211.0.0/22`)

## The Ingress controller events complain about quota, how do I increase it?

GLBC is not aware of your GCE quota. As of this writing users get 3
[GCE Backend Services](https://cloud.google.com/compute/docs/load-balancing/http/backend-service)
by default. If you plan on creating Ingresses for multiple Kubernetes Services,
remember that each one requires a backend service, and request quota. Should you
fail to do so the controller will poll periodically and grab the first free
backend service slot it finds. You can view your quota:

```console
$ gcloud compute project-info describe --project myproject
```
See [GCE documentation](https://cloud.google.com/compute/docs/resource-quotas#checking_your_quota)
for how to request more.

## Why does the Ingress need a different instance group then the GKE cluster?

The controller adds/removes Kubernets nodes that are `NotReady` from the lb
instance group.

## Why does the cloud console show 0/N healthy instances?

Some nodes are reporting negatively on the GCE HTTP health check.
Please check the following:
1. Try to access any node-ip:node-port/health-check-url
2. Try to access any pubic-ip:node-port/health-check-url
3. Make sure you have a firewall-rule allowing access to the GCE LB IP range
   (created by the Ingress controller on your behalf)
4. Make sure the right NodePort is opened in the Backend Service, and
   consequently, plugged into the lb instance group

## Can I configure GCE health checks through the Ingress?

Currently health checks are not exposed through the Ingress resource, they're
handled at the node level by Kubernetes daemons (kube-proxy and the kubelet).
However the GCE HTTP lb still requires a HTTP health check to measure node
health. By default, this health check points at `/` on the nodePort associated
with a given backend. Note that the purpose of this health check is NOT to
determine when endpoint pods are overloaded, but rather, to detect when a
given node is incapable of proxying requests for the Service:nodePort
alltogether. Overloaded endpoints are removed from the working set of a
Service via readiness probes conducted by the kubelet.

If `/` doesn't work for your application, you can have the Ingress controller
program the GCE health check to point at a readiness probe as shows in [this](/examples/health-checks/)
example.

We plan to surface health checks through the API soon.

## Why does my Ingress have an ephemeral ip?

GCE has a concept of [ephemeral](https://cloud.google.com/compute/docs/instances-and-network#ephemeraladdress)
and [static](https://cloud.google.com/compute/docs/instances-and-network#reservedaddress) IPs. A production
website would always want a static IP, which ephemeral IPs are cheaper (both in terms of quota and cost), and
are therefore better suited for experimentation.
* Creating a HTTP Ingress (i.e an Ingress without a TLS section) allocates an ephemeral IP for 2 reasons:
   * we want to encourage secure defaults
   * static-ips have limited quota and pure HTTP ingress is often used for testing
* Creating an Ingress with a TLS section allocates a static IP
* Modifying an Ingress and adding a TLS section allocates a static IP, but the
  IP *will* change. This is a beta limitation.
* You can [promote](https://cloud.google.com/compute/docs/instances-and-network#promote_ephemeral_ip)
  an ephemeral to a static IP by hand, if required.

## Can I pre-allocate a static-ip?

Yes, please see [this](/examples/static-ip) example.

## Does updating a Kubernetes secrete update the GCE TLS certs?

Yes, expect O(30s) delay.

The controller should create a second ssl certificate suffixed with `-1` and
atomically swap it with the ssl certificate in your taret proxy, then delete
the obselete ssl certificate.

## Can I tune the loadbalancing algorithm?

Right now, a kube-proxy nodePort is a necessary condition for Ingress on GCP.
This is because the cloud lb doesn't understand how to route directly to your
pods. Incorporating kube-proxy and cloud lb algorithms so they cooperate
toward a common goal is still a work in progress. If you really want fine
grained control over the algorithm, you should deploy the [nginx controller](/examples/deployment/nginx).

## Is there a maximum number of Endpoints I can add to the Ingress?

This limit is directly related to the maximum number of endpoints allowed in a
Kubernetes cluster, not the the HTTP LB configuration, since the HTTP LB sends
packets to VMs. Ingress is not yet supported on single zone clusters of size >
1000 nodes ([issue](https://github.com/kubernetes/contrib/issues/1724)). If
you'd like to use Ingress on a large cluster, spread it across 2 or more zones
such that no single zone contains more than a 1000 nodes. This is because there
is a [limit](https://cloud.google.com/compute/docs/instance-groups/creating-groups-of-managed-instances)
to the number of instances one can add to a single GCE Instance Group. In a
multi-zone cluster, each zone gets its own instance group.

## How do I match GCE resources to Kubernetes Services?

The format followed for creating resources in the cloud is:
`k8s-<resource-name>-<nodeport>-<cluster-hash>`, where `nodeport` is the output of
```console
$ kubectl get svc <svcname> --template '{{range $i, $e := .spec.ports}}{{$e.nodePort}},{{end}}'
```

`cluster-hash` is the output of:
```console
$ kubectl get configmap -o yaml --namespace=kube-system | grep -i " data:" -A 1
  data:
    uid: cad4ee813812f808
```

and `resource-name` is a short prefix for one of the resources mentioned [here](#what-are-the-cloud-resources-created-for-a-single-ingress)
(eg: `be` for backends, `hc` for health checks). If a given resource is not tied
to a single `node-port`, its name will not include the same.

## Can I change the cluster UID?

The Ingress controller configures itself to add the UID it stores in a configmap in the `kube-system` namespace.

```console
$ kubectl --namespace=kube-system get configmaps
NAME          DATA      AGE
ingress-uid   1         12d

$ kubectl --namespace=kube-system get configmaps -o yaml
apiVersion: v1
items:
- apiVersion: v1
  data:
    uid: UID
  kind: ConfigMap
...
```

You can pick a different UID, but this requires you to:

1. Delete existing Ingresses
2. Edit the configmap using `kubectl edit`
3. Recreate the same Ingress

After step 3 the Ingress should come up using the new UID as the suffix of all cloud resources. You can't simply change the UID if you have existing Ingresses, because
renaming a cloud resource requires a delete/create cycle that the Ingress controller does not currently automate. Note that the UID in step 1 might be an empty string,
if you had a working Ingress before upgrading to Kubernetes 1.3.

__A note on setting the UID__: The Ingress controller uses the token `--` to split a machine generated prefix from the UID itself. If the user supplied UID is found to
contain `--` the controller will take the token after the last `--`, and use an empty string if it ends with `--`. For example, if you insert `foo--bar` as the UID,
the controller will assume `bar` is the UID. You can either edit the configmap and set the UID to `bar` to match the controller, or delete existing Ingresses as described
above, and reset it to a string bereft of `--`.

## Why do I need a default backend?

All GCE URL maps require at least one [default backend](https://cloud.google.com/compute/docs/load-balancing/http/url-map#url_map_simplest_case), which handles all
requests that don't match a host/path. In Ingress, the default backend is
optional, since the resource is cross-platform and not all platforms require
a default backend. If you don't specify one in your yaml, the GCE ingress
controller will inject the default-http-backend Service that runs in the
`kube-system` namespace as the default backend for the GCE HTTP lb allocated
for that Ingress resource.

## How does Ingress work across 2 GCE clusters?

See federation [documentation](http://kubernetes.io/docs/user-guide/federation/federated-ingress/).

## I shutdown a cluster without deleting all Ingresses, how do I manually cleanup?

If you kill a cluster without first deleting Ingresses, the resources will leak.
If you find yourself in such a situation, you can delete the resources by hand:

1. Navigate to the [cloud console](https://console.cloud.google.com/) and click on the "Networking" tab, then choose "LoadBalancing"
2. Find the loadbalancer you'd like to delete, it should have a name formatted as: k8s-um-ns-name--UUID
3. Delete it, check the boxes to also casade the deletion down to associated resources (eg: backend-services)
4. Switch to the "Compute Engine" tab, then choose "Instance Groups"
5. Delete the Instance Group allocated for the leaked Ingress, it should have a name formatted as: k8s-ig-UUID

We plan to fix this [soon](https://github.com/kubernetes/kubernetes/issues/16337).

## How do I disable the GCE Ingress controller?

3 options:
1. Have it no-op based on the `ingress.class` annotation as shown [here](README.md#how-do-i-disable-the-ingress-controller)
2. SSH into the GCE master node and delete the GLBC manifest file found at `/etc/kubernetes/manifests/glbc.manifest`
3. Create the GKE cluster without it:

```console
$ gcloud container clusters create mycluster --network "default" --num-nodes 1 \
--machine-type n1-standard-2 --zone $ZONE \
--disable-addons HttpLoadBalancing \
--disk-size 50 --scopes storage-full
```


