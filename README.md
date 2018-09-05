# Cloud Foundry Container Runtime
A [BOSH](http://bosh.io/) release for [Kubernetes](http://kubernetes.io).  Formerly named **kubo**.

- **Slack**: #cfcr on https://slack.cloudfoundry.org
- **Pivotal Tracker**: https://www.pivotaltracker.com/n/projects/2093412

This is a fork for evolving Azure support pending PRs to kubo-release and kubo-deployment.

Note: Some form of Azure support is expected to be committed to the upstream in Fall of 2018, which may or may not include content from these repositories. This is an unoffical fork for use by users that wish to pursue and explore CFCR on Azure before the upstream supports it.

## Prerequisites
- A BOSH Director configured with UAA, Credhub, and BOSH DNS.   If this sounds annoying, the easiest single-command option is [BOSH Bootloader](https://github.com/cloudfoundry/bosh-bootloader).  
- **For Azure users**, I recommend enabling Managed Disks in your BOSH director via the [appropriate ops file](https://raw.githubusercontent.com/cloudfoundry/bosh-deployment/master/azure/use-managed-disks.yml).  
- Set BOSH DNS to Azure standard **168.63.129.16** via the [dns ops file](https://github.com/cloudfoundry/bosh-deployment/blob/master/misc/dns.yml) as the Azure Kubernetes cloud provider requires Azure private DNS domains *(\*.internal.cloudapp.net)* to be resolvable.
- **If you must bring-your-own DNS**, ping me on Slack (@svrc on CF slack, @scharlton on Pivotal slack) to work out the details.   In short, you need to support DDNS and add a DHCP client exit hook to run `nsupdate` against your DNS server (e.g. [this article from Microsoft](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-name-resolution-ddns)).   Each K8s node hostname must be DNS resolvable, and **must also be the Azure VM name** which is also the BOSH Agent ID. 
- [azure-kubo-release](https://github.com/svrc-pivotal/azure-kubo-release)
- [azure-kubo-deployment](https://github.com/svrc-pivotal/azure-kubo-deployment)
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

#### Hardware Requirements
Kubernetes uses etcd as its datastore. The official infrastructure requirements and example configurations for the etcd cluster can be found [here](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md).

## Deploying CFCR

1. Upload the [latest Xenial stemcell](https://github.com/svrc-pivotal/azure-kubo-deployment/releases/download/stemcell-98.0/bosh-stemcell-98.0-azure-hyperv-ubuntu-xenial-go_agent.tgz) to the director.  Currently for Azure users this is 98.0 in this repo's [releases](https://github.com/svrc-pivotal/azure-kubo-deployment/releases), which is a custom stemcell including the latest [BOSH agent fix](https://github.com/cloudfoundry/bosh-agent/pull/174) thanks to [andyliuliming](https://github.com/andyliuliming) from Microsoft.  
1. Upload the latest kubo-release to the director.
    ```
    bosh upload-release https://github.com/svrc-pivotal/azure-kubo-deployment/releases/download/0.20.0.azure%2Bdev.2/kubo-release-0.20.0.azure+dev.2.tgz
    ```

1. Deploy

    ##### Option 1. Single Master

	```bash
	cd kubo-deployment

	bosh deploy -d cfcr manifests/cfcr.yml \
	  -o manifests/ops-files/misc/single-master.yml \
	  -o manifests/ops-files/add-hostname-to-master-certificate.yml \
	  -v api-hostname=[DNS-NAME]
	```

    ##### Option 2. Three Masters

1. Upload the [latest Xenial stemcell](https://github.com/svrc-pivotal/azure-kubo-deployment/releases/download/stemcell-98.0/bosh-stemcell-98.0-azure-hyperv-ubuntu-xenial-go_agent.tgz) to the director. Currently for Azure users this is 98.0 in this repo's releases, which is a custom stemcell including the latest BOSH agent fix thanks to [andyliuliming](https://github.com/andyliuliming) from Microsoft.
1. Upload the latest kubo-release to the director.
    ```
    bosh upload-release https://github.com/svrc-pivotal/azure-kubo-deployment/releases/download/0.20.0.azure%2Bdev.2/kubo-release-0.20.0.azure+dev.2.tgz
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

   Check additional configurations, such as setting Kubernetes cloud provider, in [docs](./docs/cloud-provider.md).

1. Add Kubernetes system components

    ```bash
    bosh -d cfcr run-errand apply-specs
    ```

1. Run the following to confirm the cluster is operational
	```
	bosh -d cfcr run-errand smoke-tests
	```
## Adding the Azure Cloud Config
1. Deploy (or re-deploy) with the following options, providing an azure.yml variable file for the cloud provider variables.
	```
	bosh deploy -d cfcr manifests/cfcr.yml \
	  -o manifests/ops-files/add-vm-extensions-to-master.yml \
	  -o manifests/ops-files/add-hostname-to-master-certificate.yml \
	  -o manifests/ops-files/iaas/azure/cloud-provider.yml
	  -v api-hostname=[LOADBALANCER-ADDRESS]
	  -l azure.yml
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

## Monitoring

Follow the recommendations in [etcd's documentation](https://github.com/etcd-io/etcd/blob/master/Documentation/metrics.md) for monitoring etcd
metrics.

## Deprecations

### CFCR Docs
We are no longer supporting the following documentation for deploying BOSH and CFCR
* https://docs-cfcr.cfapps.io

The [deploy_bosh](https://github.com/svrc-pivotal/azure-kubo-deployment/blob/master/bin/deploy_bosh)
and [deploy_k8s](https://github.com/svrc-pivotal/azure-kubo-deployment/blob/master/bin/deploy_k8s)
scripts in the `kubo-deployment` repository are now deprecated.

### Heapster
K8s 1.11 release kicked off the deprecation timeline for the Heapster component, see [here](https://github.com/kubernetes/heapster/blob/master/docs/deprecation.md) for more info. As a result, we're in the process of replacing Heapster with [Metrics Server](https://github.com/kubernetes-incubator/metrics-server) in the upcoming releases of kubo-release.
