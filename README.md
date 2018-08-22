# Cloud Foundry Container Runtime
A [BOSH](http://bosh.io/) release for [Kubernetes](http://kubernetes.io).  Formerly named **kubo**.

- **Slack**: #cfcr on https://slack.cloudfoundry.org
- **Pivotal Tracker**: https://www.pivotaltracker.com/n/projects/2093412

## Prerequisites
- A BOSH Director configured with UAA, Credhub, and BOSH DNS.
- [kubo-release](https://github.com/cloudfoundry-incubator/kubo-release)
- [kubo-deployment](https://github.com/cloudfoundry-incubator/kubo-deployment)
- Accessing the master:
  - **Single Master:** Set up a DNS name pointing to your master's IP address
  - **Multiple Masters:** A TCP load balancer for your master nodes.
    - Use a TCP load balancer configured to connect to the master nodes on port 8443.
    - Add healthchecks using either a TCP dial or HTTP by looking for a `200 OK` response from `/healthz`.
- Cloud Config with
  - `vm_types` named `minimal`, `small`, and `small-highmem` (See [cf-deployment](https://github.com/cloudfoundry/cf-deployment) for reference)
  - `network` named `default`
  - There are three availability zones `azs`, and they are named `z1`,`z2`,`z3`
  - Note: the cloud-config properties can be customized by applying ops-files. See `manifests/ops-files` for some examples
  - If using loadbalancers then apply this `vm_extension` called `cfcr-master-loadbalancer` to the cloud-config to add the instances to your loadbalancers. See [BOSH documentation](https://bosh.io/docs/cloud-config/#vm-extensions) for information on how to configure loadbalancers.

## Deploying CFCR

### Single Master

1. Upload the [latest Xenial stemcell](https://bosh.io/stemcells/#ubuntu-xenial) to the director.
1. Upload the latest kubo-release to the director.
    ```
    bosh upload-release https://bosh.io/d/github.com/cloudfoundry-incubator/kubo-release
    ```
1. Deploy
	```
	cd kubo-deployment

	bosh deploy -d cfcr manifests/cfcr.yml \
	  -o manifests/ops-files/misc/single-master.yml \
	  -o manifests/ops-files/add-hostname-to-master-certificate.yml \
	  -v api-hostname=[DNS-NAME]
	```
1. Add kubernetes system components
	```
	bosh -d cfcr run-errand apply-specs
	```
1. Run the following to confirm the cluster is operational
	```
	bosh -d cfcr run-errand smoke-tests
	```

### Three Masters

1. Upload the [latest Xenial stemcell](https://bosh.io/stemcells/#ubuntu-xenial) to the director.
1. Upload the latest kubo-release to the director.
    ```
    bosh upload-release https://bosh.io/d/github.com/cloudfoundry-incubator/kubo-release
    ```
1. Deploy
	```
	cd kubo-deployment

	bosh deploy -d cfcr manifests/cfcr.yml \
	  -o manifests/ops-files/add-vm-extensions-to-master.yml \
	  -o manifests/ops-files/add-hostname-to-master-certificate.yml \
	  -v api-hostname=[LOADBALANCER-ADDRESS]
	```

  *Note: Loadbalancer address should be the external address (hostname or IP) of the loadbalancer you have configured.*
1. Add kubernetes system components
	```
	bosh -d cfcr run-errand apply-specs
	```
1. Run the following to confirm the cluster is operational
	```
	bosh -d cfcr run-errand smoke-tests
	```

### BOSH Lite
CFCR clusters on BOSH Lite are intended for development. We run the [deploy_cfcr_lite](https://github.com/cloudfoundry-incubator/kubo-deployment/blob/master/bin/deploy_cfcr_lite) script to provision a cluster with the latest stemcell and master of kubo-release.

```
cd kubo-deployment
./bin/deploy_cfcr_lite
```
## Accessing the CFCR Cluster with kubectl

1. Login to the Credhub Server that stores the cluster's credentials:
	```
	credhub login
	```
1. Find the director name by running
	```
	bosh env
	```
1. Configure the `kubeconfig` for your `kubectl` client:
	```
	cd kubo-deployment

	./bin/set_kubeconfig <DIRECTOR_NAME>/cfcr https://[DNS-NAME-OR-LOADBALANCER-ADDRESS]:8443
	```

## Deprecations

### CFCR Docs
We are no longer supporting the following documentation for deploying BOSH and CFCR
* https://docs-cfcr.cfapps.io

The [deploy_bosh](https://github.com/cloudfoundry-incubator/kubo-deployment/blob/master/bin/deploy_bosh)
and [deploy_k8s](https://github.com/cloudfoundry-incubator/kubo-deployment/blob/master/bin/deploy_k8s)
scripts in the `kubo-deployment` repository are now deprecated.

### Heapster
K8s 1.11 release kicked off the deprecation timeline for the Heapster component, see [here](https://github.com/kubernetes/heapster/blob/master/docs/deprecation.md) for more info. As a result, we're in the process of replacing Heapster with [Metrics Server](https://github.com/kubernetes-incubator/metrics-server) in the upcoming releases of kubo-release.
