---
authors: Dean Roehrich <dean.roehrich@@hpe.com>
categories: install
---

# DWS On CSM: Installation and use of DWS on a CSM cluster

DataWorkflowServices (DWS) on CSM consists of two parts.  The first part is the DWS service itself, which must be installed on the CSM cluster.  The second part is the Slurm Burst Buffer Plugin which must be installed on the system where Slurm is running. The Burst Buffer Plugin is used by Slurm to talk to the DWS API.

## Prepare For Install Or Upgrade of DWS

If DWS is not yet installed on the CSM cluster, then proceed to the **Install DWS** section.

If DWS is already installed on the CSM cluster then that version must be undeployed before the next version can be installed.  This is necessary to properly handle updates to the DWS Custom Resource Definitions (CRDs).  Proceed to the **Undeploy DWS** section before returning here to install the new version.

## Install DWS


### Retrieve DWS Configuration and Container Image

The DWS Configuration is in the DWS repo.  It is not necessary to build DWS, but this is where its configuration files will be found.  These include the DWS CRDs, Deployment, ServiceAccount, Roles and bindings, Services, and Secrets.

```console title="ncn-m001:~ #"
DWS_VER=0.0.20
git clone --branch v$DWS_VER https://github.com/DataWorkflowServices/dws.git dws-$DWS_VER
cd dws-$DWS_VER
```

This workarea must be preserved.  It will be used again when this version of DWS must be undeployed.

### Load the DWS Container Images

DWS uses two container images which must be loaded into the CSM cluster's container registry. The Nexus Admin Credential will be used to push these images into the registry.

Obtain the Nexus Admin user name.  Store it in a variable to be used in later commandlines:

```console title="ncn-m001:~/dws #"
NEXUS_USER=$(kubectl get secret -n nexus nexus-admin-credential -o json | jq .data.username -r | base64 -d)
```

Obtain the Nexus Admin password.  Keep this output available so it can be used with cut-and-paste when podman requests the password:

```console title="ncn-m001:~/dws #"
kubectl get secret -n nexus nexus-admin-credential -o json | jq .data.password -r | base64 -d
```

Use the Nexus Admin user name and password to load the container images into the container registry.  Adjust or replace the following `podman pull` commands as necessary, depending on your network restrictions, to copy each container in from `ghcr.io`.

Get the DWS container image corresponding to this repo:

```console title="ncn-m001:~/dws #"
podman pull ghcr.io/dataworkflowservices/dws:$DWS_VER
podman tag ghcr.io/dataworkflowservices/dws:$DWS_VER registry.local/dws:$DWS_VER
podman push --creds $NEXUS_USER registry.local/dws:$DWS_VER
```

Get the kube-rbac-proxy container:

```console title="ncn-m001:~ #"
RBAC_PROXY_VER=v0.14.1
podman pull docker://gcr.io/kubebuilder/kube-rbac-proxy:$RBAC_PROXY_VER
podman tag gcr.io/kubebuilder/kube-rbac-proxy:$RBAC_PROXY_VER registry.local/kube-rbac-proxy:$RBAC_PROXY_VER
podman push --creds $NEXUS_USER registry.local/kube-rbac-proxy:$RBAC_PROXY_VER
```

### Deploy DWS to the cluster

Deploy DWS to the cluster:

```console title="ncn-m001:~/dws #"
make kustomize
make deploy OVERLAY=csm
```

Wait for the deployment and webhook to become ready.

```console title="ncn-m001:~/dws #"
kubectl wait deployment --timeout=120s -n dws-system dws-controller-manager --for condition=Available=True
kubectl wait deployment --timeout=120s -n dws-system dws-webhook --for condition=Available=True
```

## Undeploy DWS

All Slurm jobs must be completed before DWS can be undeployed.  In addition, all DWS Workflow resources must have been deleted.

Use the same workarea that was used to install DWS.  It should still be set to the same version of DWS that will be undeployed.

Confirm that all DWS Workflow resources have been deleted:

```console title="ncn-m001:~/dws #"
kubectl get workflows.dataworkflowservices.github.io -A
No resources found
```

If any workflows remain, then some Slurm jobs are not yet completed.  All Slurm jobs must be completed before DWS can be undeployed.

To undeploy DWS:

```console title="ncn-m001:~/dws #"
make undeploy OVERLAY=csm
```

Ignore any errors about not finding secrets.  They were removed by garbage collection during earlier steps in the processing of `kubectl delete`.

## RBAC: Role-Based Access Control

RBAC (Role Based Access Control) determines the operations a user or service can perform on a list of Kubernetes resources. RBAC affects everything that interacts with the kube-apiserver (both users and services internal or external to the cluster). More information about RBAC can be found in the Kubernetes [***documentation***](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

This is a Slurm-specific version of the instructions found in [RBAC: Role-Based Access Control](https://nearnodeflash.github.io/guides/rbac-for-users/readme/) for interacting with DWS. Consult that document for more details.

### RBAC for Burst Buffer Plugin

A workload manager (WLM) such as Slurm will interact with DataWorkflowServices as a privileged user. RBAC is used to limit the operations that a WLM can perform:

- Generate a new key/cert pair for a "slurm-dws" user
- Creating a new kubeconfig file
- Adding RBAC rules for the "slurm-dws" user to allow appropriate access to the DataWorkflowServices API.

**Note** Each of these steps must be performed on the CSM management node.

### Generate a Key and Certificate

The first step is to create a new key and certificate for the "slurm-dws" user.  This will likely be done on one of the CSM management nodes. The `openssl` command needs access to the certificate authority file. This is typically located in `/etc/kubernetes/pki`.

```console title="ncn-m001:~/dws #"
# make a temporary work space
mkdir /tmp/slurm-dws
cd /tmp/slurm-dws

# Create this user
export USERNAME=slurm-dws

# generate a new key
openssl genrsa -out slurm-dws.key 2048

# create a certificate signing request for this user
openssl req -new -key slurm-dws.key -out slurm-dws.csr -subj "/CN=$USERNAME"

# generate a certificate using the certificate authority on the k8s cluster. This certificate lasts 500 days
openssl x509 -req -in slurm-dws.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out slurm-dws.crt -days 500
```

### Create a kubeconfig

After the keys have been generated, a new kubeconfig file can be created for this user. The admin kubeconfig `/etc/kubernetes/admin.conf` can be used to determine the cluster name kube-apiserver address.

```console title="ncn-m001:~/dws #"
# get the cluster name and server
CURRENT_CONTEXT=$(kubectl config current-context)
CLUSTER_NAME=$(kubectl config get-contexts | grep $CURRENT_CONTEXT | awk '{print $3}')
SERVER=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "'$CLUSTER_NAME'")].cluster.server}')

# create a new kubeconfig with the server information
kubectl config set-cluster $CLUSTER_NAME --kubeconfig=slurm-dws.kubeconfig --server=$SERVER --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true

# add the key and cert for this user to the config
kubectl config set-credentials $USERNAME --kubeconfig=slurm-dws.kubeconfig --client-certificate=slurm-dws.crt --client-key=slurm-dws.key --embed-certs=true

# add a context
kubectl config set-context $USERNAME --kubeconfig=slurm-dws.kubeconfig --cluster=$CLUSTER_NAME --user=$USERNAME

# make it the default context
kubectl config use-context $USERNAME --kubeconfig slurm-dws.kubeconfig
```

**Important** This new kubeconfig file must be installed as `/etc/slurm/slurm-dws.kubeconfig` on the system that has slurmctld and the burst buffer plugin.

### Apply the provided ClusterRole and create a ClusterRoleBinding

DataWorkflowServices has already defined the role to be used with WLMs.  Simply apply the `workload-manager` ClusterRole from DataWorkflowServices to the system, found in the dws repo that was checked out earlier:

```console title="ncn-m001:~/dws #"
kubectl apply -f config/rbac/workload_manager_role.yaml
```

Create and apply a ClusterRoleBinding to associate the "slurm-dws" user with the `workload-manager` ClusterRole:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: slurm-dws-viewer
subjects:
- kind: User
  name: slurm-dws
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: workload-manager
  apiGroup: rbac.authorization.k8s.io
```

That resource can be created using the `kubectl apply` command.

## Configure Slurm for DWS

Slurm provides an API for burst buffer plugins that it uses to talk to a workload manager (WLM).  The plugin is written in the Lua programming language and is placed in the same directory that holds the rest of the Slurm configuration files.  This plugin contains the logic required to communicate with DWS.

This project will install two files into the slurm pod's `/etc/slurm` directory and will update the main Slurm configuration file to enable the plugin.  The first will be the Lua plugin script, named `burst_buffer.lua`, and the second will be a configuration file named `burst_buffer.conf` which tells Slurm that job scripts containing a `#DW` directive should be run through the burst_buffer plugin.

The burst buffer plugin will use the `kubectl` command to access the Kubernetes environment where DWS is running, so the system running the slurmctld must have this command and a valid `kubeconfig` file installed as `/etc/slurm/slurm-dws.kubeconfig`. See [RBAC: Role-Based Access Control](#rbac-role-based-access-control) above for the creation of this kubeconfig file.

### Checkout the DWS Slurm Burst Buffer Plugin repo

On the Slurm system running slurmctl, check out the repo containing the burst buffer plugin and its configuration file:

```console
BBPLUGIN_VER=0.0.5
git clone --branch v$BBPLUGIN_VER https://github.com/DataWorkflowServices/dws-slurm-bb-plugin.git dws-slurm-bb-plugin-$BBPLUGIN_VER
cd dws-slurm-bb-plugin-$BBPLUGIN_VER
```

### Install the burst buffer plugin

Copy the burst buffer plugin and its configuration file into the Slurm configuration directory from the repo you've checked out above:

```console title="~/dws-slurm-bb-plugin #"
cp src/burst_buffer/burst_buffer.lua /etc/slurm
cp src/burst_buffer/burst_buffer.conf /etc/slurm
```

### Enable the burst buffer plugin

On the system running slurmctld, edit `/etc/slurm/slurm.conf` to enable the use of the burst buffer plugin.

```console title="~/dws-slurm-bb-plugin #"
echo "BurstBufferType=burst_buffer/lua" >> /etc/slurm/slurm.conf
```

Then restart slurmctld to pick up the new configuration.

## Prepare to submit a test job

From the Slurm system that has slurmctld, where the burst buffer plugin will run, verify that the `/etc/slurm/slurm-dws.kubeconfig` file that was created earlier is available by verifying that it can be used to reach the DWS API:

```console
kubectl --kubeconfig /etc/slurm/slurm-dws.kubeconfig get workflows.dataworkflowservices.github.io -A
```

There should be no workflows and you should see only the following output:

```console
No resources found
```

## Submit a test job

Create the following example test job:

```conf title="$ cat /tmp/dws-test" linenums="1"
#!/bin/sh                
#SBATCH --time=1
#DW
/bin/hostname
srun -l /bin/hostname
srun -l /bin/pwd
```

Submit the test job:

```console
sbatch /tmp/dws-test
```

Get the DWS Workflow via `kubectl`, running from the Slurm system that has slurmctld.  If this step fails then return to [Prepare to submit a test job](#prepare-to-submit-a-test-job) to verify the correct kubectl configuration.

```console
kubectl --kubeconfig /etc/slurm/slurm-dws.kubeconfig get workflows.dataworkflowservices.github.io -A
```

Get the status of the DWS Workflow via `scontrol`.  This will cause Slurm to use the burst buffer plugin to access the DWS API, and will use `kubectl` under the covers:

```console
scontrol show bbstat workflow 7773 && echo
```

The output will appear as:

```console
desiredState=DataIn currentState=DataIn status=Completed
```

Note that the workflow resource will no longer exist after Slurm has completed its teardown state for this job.

## Canceling a test job

Use of the `scancel` command in states prior to PostRun will cause Slurm to pass the `hurry` flag to the teardown function in the burst_buffer.lua script, and the teardown function will set the flag in the Workflow resource.  In the PostRun or DataOut states the burst_buffer.lua teardown function will not be passed the `hurry` flag because no work will be skipped.  In all states, the `scancel --hurry` command will cause Slurm to pass the `hurry` flag to the teardown function.

The `scancel` command will cause states prior to PostRun to terminate immediately and proceed to Teardown state.  The use of `scancel --hurry` does not alter this behavior on these pre-PostRun states.  During PostRun or DataOut, the `scancel` command does not cause early termination and does not skip the DataOut state.  The use of `scancel --hurry` during PostRun or DataOut causes early termination, skipping DataOut in the case of PostRun, and proceeds to Teardown.

Consult the [Slurm Burst Buffer Guide](https://slurm.schedmd.com/burst_buffer.html) for further details on the use of `scancel` versus `scancel --hurry`.

## Troubleshooting

### Collect slurmctld logs

To pick out the burst buffer plugin messages from slurmctld logs, grep for the string "lua".

### Collect DWS logs

The DWS logs can be retrieved with the following.  These commands should be run from the CSM management node.  Run the `kubectl` command using the adminstrator context:

```console title="ncn-m001:~ #"
DWS_POD=`kubectl --context kubernetes-admin@kubernetes get pods -n dws-system -l control-plane=controller-manager -o jsonpath='{.items[0].metadata.name}'`
kubectl --context kubernetes-admin@kubernetes logs -n dws-system $DWS_POD -c manager
```

### Inspect DWS Workflow resources

Inspect DWS Workflow resources by using the fully-qualified CRD name.

```console title="ncn-m001:~ #"
kubectl --context kubernetes-admin@kubernetes get workflows.dataworkflowservices.github.io -A
```

The output will show the job ID used in the name of the Workflow resource:

```console
NAMESPACE   NAME     STATE    READY   STATUS      AGE
user        bb7773   DataIn   true    Completed   13m
```


To get more detail about the workflow while it exists:

```console title="ncn-m001:~ #"
kubectl --context kubernetes-admin@kubernetes get workflow.dataworkflowservices.github.io -n user bb7773 -o yaml
```

Note that the Workflow resource will be deleted during the job's teardown state, at which time it will no longer appear in the workflow listing.

<details>
<summary>Multiple CRDs named "Workflow"</summary>
    The fully-qualified name will protect you from picking up any resource named "Workflow" that belongs to some other service.

Your system may have multiple CRDs named "Workflow".  To see the CRDs named "Workflow":

```console title="ncn-m001:~ #"
kubectl --context kubernetes-admin@kubernetes get crds | grep -E '^workflows\.'
```

Example output:

```console
workflows.argoproj.io                                               2022-09-16T14:21:54Z
workflows.dataworkflowservices.github.io                            2023-09-28T19:20:23Z
```
</details>

## References

* [Slurm Burst Buffer Guide](https://slurm.schedmd.com/burst_buffer.html) 
* [Slurm burst_buffer.conf manage](https://slurm.schedmd.com/burst_buffer.conf.html)

