---
authors: Dean Roehrich <dean.roehrich@@hpe.com>
categories: install
---

# DWS On CSM: Installation and use of DWS on a CSM cluster

## Install DWS

### DWS Dependencies

Install the DWS dependencies and set the expected node labels prior to installing DWS.

<details>
<summary>DWS Dependencies</summary>
    DWS requires kube-rbac-proxy to be present in the cluster's container registry.

```console title="ncn-m001:~ #"
podman run --rm --network host quay.io/skopeo/stable:v1.4.1 copy --dest-tls-verify=false docker://gcr.io/kubebuilder/kube-rbac-proxy:v0.13.0 docker://registry.local/kube-rbac-proxy:v0.13.0
```

In the past, we asked that a worker node be labelled with `cray.wlm.manager`.  DWS no longer uses this label, and you may remove it.
</details>

### Retrieve DWS Configuration and Container Image

The DWS Configuration is in the DWS repo.  We don't need to build DWS here, but we do need its configuration files from the repo.  This is where we will get the DWS CRDs, Deployment, ServiceAccount, Roles and bindings, Services, and Secrets.

```console title="ncn-m001:~ #"
DWS_VER=0.0.11
git clone --branch v$DWS_VER https://github.com/DataWorkflowServices/dws.git
cd dws
```

Get the DWS container image corresponding to this repo.  This must be made present in the cluster's container registry.

```console title="ncn-m001:~/dws #"
podman run --rm --network host quay.io/skopeo/stable:v1.4.1 copy --dest-tls-verify=false docker://ghcr.io/dataworkflowservices/dws:$DWS_VER docker://registry.local/dws:$DWS_VER
```

### Deploy DWS to the cluster

Deploy DWS to the cluster.  Set `IMAGE_TAG_BASE` to point at the DWS image in the cluster's container registry from the previous step.  Set the `OVERLAY=csm` to pick the configuration for CSM clusters.

```console title="ncn-m001:~/dws #"
IMAGE_TAG_BASE=registry.local/dws OVERLAY=csm make deploy
```

Wait for the deployment and webhook to become ready.

```console title="ncn-m001:~/dws #"
kubectl wait deployment --timeout=120s -n dws-operator-system dws-operator-controller-manager --for condition=Available=True
kubectl wait deployment --timeout=120s -n dws-operator-system dws-operator-webhook --for condition=Available=True
```

To undeploy DWS, after all Workflow resources have been deleted:

```console title="ncn-m001:~/dws #"
IMAGE_TAG_BASE=registry.local/dws OVERLAY=csm make undeploy
```

## Configure Slurm for DWS

Slurm provides an API for burst buffer plugins that it uses to talk to a workload manager (WLM).  The plugin is written in the Lua programming language and is placed in the same directory that holds the rest of the Slurm configuration files.  This plugin contains the logic required to communicate with DWS.

This project will install two files into the slurm pod's `/etc/slurm` directory and will update the main Slurm configuration file to enable the plugin.  The first will be the Lua plugin script, named `burst_buffer.lua`, and the second will be a configuration file named `burst_buffer.conf` which tells Slurm that job scripts containing a `#DW` directive should be run through the burst_buffer plugin.

### HPE Cray Programming Environment Installation Guide

The following instructions use section 10.3.2 in [HPE Cray Programming Environment Installation Guide: CSM on HPE Cray EX Systems (22.10) S-8003](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=a00126783en_us).

### Checkout the DWS repo for Slurm plugins.

```console title="ncn-m001:~ #"
git clone https://github.com/DataWorkflowServices/dws-slurm-bb-plugin.git
```

### Update the Slurm Configuration Template

The changes to the Slurm pod's `/etc/slurm` directory are controlled through a ConfigMap.  These instructions follow section 10.3.2 in [HPE Cray Programming Environment Installation Guide: CSM on HPE Cray EX Systems (22.10) S-8003](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=a00126783en_us) to update and activate the ConfigMap.

Get the slurm-config-templates ConfigMap.

```console title="ncn-m001:~ #"
kubectl get configmap -n services slurm-config-templates -o yaml > slurm-config-templates.yaml
```

Extract the `slurm.conf` file, update it to enable the burst buffer plugin, and write it back into the ConfigMap.

```console title="ncn-m001:~ #"
yq r slurm-config-templates.yaml 'data."slurm.conf"' > slurm.conf
echo "BurstBufferType=burst_buffer/lua" >> slurm.conf
yq w -i slurm-config-templates.yaml 'data."slurm.conf"' "$(cat slurm.conf)"
```

Add the `burst_buffer.lua` plugin script and `burst_buffer.conf` configuration file to the ConfigMap.

```console title="ncn-m001:~ #"
yq w -i slurm-config-templates.yaml -- 'data."burst_buffer.lua"' "$(cat dws-slurm-bb-plugin/src/burst_buffer/burst_buffer.lua)"
yq w -i slurm-config-templates.yaml 'data."burst_buffer.conf"' "$(cat dws-slurm-bb-plugin/src/burst_buffer/burst_buffer.conf)"
```

Apply the updated ConfigMap resource.

```console title="ncn-m001:~ #"
kubectl apply -f slurm-config-templates.yaml
```

### Restart the Slurm Configuration Job

The Slurm configuration job will process the ConfigMap into a second one that is included in the pod spec.  The following steps to reconfigure Slurm are found in the HPE Cray Programming Installation Guide.

```console title="ncn-m001:~ #"
JOBNAME=$(kubectl get job -n services | grep slurm-config | grep -v import | awk '{print $1}')
kubectl get job -n services -o yaml $JOBNAME > slurm-config.yaml
kubectl delete -f slurm-config.yaml
yq d -i slurm-config.yaml spec.template.metadata
yq d -i slurm-config.yaml spec.selector
kubectl apply -f slurm-config.yaml
SLURMCTLD_POD=$(kubectl get pod -n user -lapp=slurmctld -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n user $SLURMCTLD_POD -c slurmctld -- scontrol reconfigure
```

After the configuration job has been restarted, wait about 30 seconds to see the new content appear inside the pod.

If you were only adding or modifying burst_buffer.lua, then Slurm is now ready for your jobs.  If you were also modifying slurm.conf to set the BurstBufferType, or you were adding or modifying burst_buffer.conf, then you must also restart the Slurm deployments.

```console title="ncn-m001:~ #"
kubectl rollout restart deployment -n user slurmctld-backup
kubectl wait deployment --timeout=60s -n user --for condition=Available=True -l app=slurmctld-backup
kubectl rollout restart deployment -n user slurmctld
kubectl wait deployment --timeout=60s -n user --for condition=Available=True -l app=slurmctld
```

## Submit a test job

Use a UAN host to submit a test job.  The following assumes a UAN host named `uan01` and a user account named `vers`.

```console
ncn-m001:~ # ssh uan01
uan01:~ # su vers
vers@uan01:/root> cd /lus/vers
```

Create the following example test job:

```conf title="vers@uan01:/lus/vers> cat /tmp/dws-test" linenums="1"
#!/bin/sh                
#SBATCH --time=1
#DW
/bin/hostname
srun -l /bin/hostname
srun -l /bin/pwd
```

Note that we will submit the test job from a directory where our `vers` account can write its output files.  The output file location can be controlled by the `sbatch` command or by an `SBATCH` directive in the job script.


```console title="vers@uan01:/lus/vers>"
sbatch /tmp/dws-test
```

The sbatch command will tell you the ID of your job.
```console
Submitted batch job 7773
```

You can use that to query the job's status.  In this example the job's output will be in the file named `slurm-<ID>` in the same directory where the `sbatch` command was executed.

```console title="vers@uan01:/lus/vers> "
sacct -b -j 7773
scontrol show job 7773
cat slurm-7773.out
```

Get the status of the workflow.

```console title="vers@uan01:/lus/vers>"
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

To collect the slurmctld logs:

```console title="ncn-m001:~ #"
SLURM_POD=`kubectl get pods -n user -l app=slurmctld -o jsonpath='{.items[0].metadata.name}'`
kubectl logs -n user $SLURM_POD slurmctld
```

To pick out the messages from the burst_buffer.lua script:

```console title="ncn-m001:~ #"
kubectl logs -n user $SLURM_POD slurmctld | grep lua:
```

### Collect DWS logs

The DWS logs can be retrieved with the following:

```console title="ncn-m001:~ #"
DWS_POD=`kubectl get pods -n dws-operator-system -l control-plane=controller-manager -o jsonpath='{.items[0].metadata.name}'`
kubectl logs -n dws-operator-system $DWS_POD -c manager
```

### Inspect DWS Workflow resources

Inspect DWS Workflow resources by using the fully-qualified CRD name.

```console title="ncn-m001:~ #"
kubectl get workflows.dws.cray.hpe.com -A
```

The output will show the job ID used in the name of the Workflow resource:

```console
NAMESPACE   NAME     STATE    READY   STATUS      AGE
user        bb7773   DataIn   true    Completed   13m
```


To get more detail about the workflow while it exists:

```console title="ncn-m001:~ #"
kubectl get workflow.dws.cray.hpe.com -n user bb7773 -o yaml
```

Note that the Workflow resource will be deleted during the job's teardown state, at which time it will no longer appear in the workflow listing.

<details>
<summary>Multiple CRDs named "Workflow"</summary>
    The fully-qualified name will protect you from picking up any resource named "Workflow" that belongs to some other service.

Your system may have multiple CRDs named "Workflow".  To see the CRDs named "Workflow":

```console title="ncn-m001:~ #"
kubectl get crds | grep -E '^workflows\.'
```

Example output:

```console
workflows.argoproj.io                            2022-11-25T02:56:48Z
workflows.dws.cray.hpe.com                       2022-11-21T15:38:13Z
```
</details>

## References

* Section 10.3.2 of [HPE Cray Programming Environment Installation Guide: CSM on HPE Cray EX Systems (22.10) S-8003](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=a00126783en_us)
* [Slurm Burst Buffer Guide](https://slurm.schedmd.com/burst_buffer.html) 
* [Slurm burst_buffer.conf manage](https://slurm.schedmd.com/burst_buffer.conf.html)

