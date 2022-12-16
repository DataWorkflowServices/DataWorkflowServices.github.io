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

```bash
ncn-m001:~ # podman run --rm --network host quay.io/skopeo/stable:v1.4.1 copy --dest-tls-verify=false docker://gcr.io/kubebuilder/kube-rbac-proxy:v0.13.0 docker://registry.local/kube-rbac-proxy:v0.13.0
```

DWS will be deployed on a kubernetes worker node labeled with `cray.wlm.manager`.

```bash
ncn-m001:~ # kubectl label node ncn-w001 cray.wlm.manager=true
```
</details>

### Retrieve DWS Configuration and Container Image

The DWS Configuration is in the DWS repo.  We don't need to build DWS here, but we do need its configuration files from the repo.  This is where we will get the DWS CRDs, Deployment, ServiceAccount, Roles and bindings, Services, and Secrets.

```bash
ncn-m001:~ # DWS_VER=0.0.6
ncn-m001:~ # git clone --branch v$DWS_VER https://github.com/HewlettPackard/dws.git
ncn-m001:~ # cd dws
```

Get the DWS container image corresponding to this repo.  This must be made present in the cluster's container registry.

```bash
ncn-m001:~/dws # podman run --rm --network host quay.io/skopeo/stable:v1.4.1 copy --dest-tls-verify=false docker://ghcr.io/hewlettpackard/dws:$DWS_VER docker://registry.local/dws:$DWS_VER
```

### Deploy DWS to the cluster

Deploy DWS to the cluster.  Set `IMAGE_TAG_BASE` to point at the DWS image in the cluster's container registry from the previous step.  Set the `OVERLAY=csm` to pick the configuration for CSM clusters.

```bash
ncn-m001:~/dws # IMAGE_TAG_BASE=registry.local/dws OVERLAY=csm make deploy
```

To undeploy DWS, after all Workflow resources have been deleted:

```bash
ncn-m001:~/dws # IMAGE_TAG_BASE=registry.local/dws OVERLAY=csm make undeploy
```

## Configure Slurm for DWS

Slurm provides an API for burst buffer plugins that it uses to talk to a workload manager (WLM).  The plugin is written in the Lua programming language and is placed in the same directory that holds the rest of the Slurm configuration files.  This plugin contains the logic required to communicate with DWS.

This project will install two files into the slurm pod's `/etc/slurm` directory and will update the main Slurm configuration file to enable the plugin.  The first will be the Lua plugin script, named `burst_buffer.lua`, and the second will be a configuration file named `burst_buffer.conf` which tells Slurm that job scripts containing a `#DW` directive should be run through the burst_buffer plugin.

### HPE Cray Programming Environment Installation Guide

The following instructions use section 10.3.2 in [HPE Cray Programming Environment Installation Guide: CSM on HPE Cray EX Systems (22.10) S-8003](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=a00126783en_us).

### Checkout the DWS repo for Slurm plugins.

```bash
ncn-m001:~ # git clone https://github.com/DataWorkflowServices/dws-slurm-bb-plugin.git
```

### Update the Slurm Configuration Template

The changes to the Slurm pod's `/etc/slurm` directory are controlled through a ConfigMap.  These instructions follow section 10.3.2 in [HPE Cray Programming Environment Installation Guide: CSM on HPE Cray EX Systems (22.10) S-8003](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=a00126783en_us) to update and activate the ConfigMap.

Get the slurm-config-templates ConfigMap.

```bash
ncn-m001:~ # kubectl get configmap -n services slurm-config-templates -o yaml > slurm-config-templates.yaml
```

Extract the `slurm.conf` file, update it to enable the burst buffer plugin, and write it back into the ConfigMap.

```bash
ncn-m001:~ # yq r slurm-config-templates.yaml 'data."slurm.conf"' > slurm.conf
ncn-m001:~ # echo "BurstBufferType=burst_buffer/lua" >> slurm.conf
ncn-m001:~ # yq w -i slurm-config-templates.yaml 'data."slurm.conf"' "$(cat slurm.conf)"
```

Add the `burst_buffer.lua` plugin script and `burst_buffer.conf` configuration file to the ConfigMap.

```bash
ncn-m001:~ # yq w -i slurm-config-templates.yaml 'data."burst_buffer.lua"' "$(cat dws-slurm-bb-plugin/src/burst_buffer/burst_buffer.lua)"
ncn-m001:~ # yq w -i slurm-config-templates.yaml 'data."burst_buffer.conf"' "$(cat dws-slurm-bb-plugin/src/burst_buffer/burst_buffer.conf)"
```

Apply the updated ConfigMap resource.

```bash
ncn-m001:~ # kubectl apply -f slurm-config-templates.yaml
```

### Restart the Slurm Configuration Job

The Slurm configuration job will process the ConfigMap into a second one that is included in the pod spec.  The following steps to reconfigure Slurm are found in the HPE Cray Programming Installation Guide.

```bash
ncn-m001:~ # JOBNAME=$(kubectl get job -n services | grep slurm-config | grep -v import | awk '{print $1}')
ncn-m001:~ # kubectl get job -n services -o yaml $JOBNAME > slurm-config.yaml
ncn-m001:~ # kubectl delete -f slurm-config.yaml
ncn-m001:~ # yq d -i slurm-config.yaml spec.template.metadata
ncn-m001:~ # yq d -i slurm-config.yaml spec.selector
ncn-m001:~ # kubectl apply -f slurm-config.yaml
ncn-m001:~ # SLURMCTLD_POD=$(kubectl get pod -n user -lapp=slurmctld -o jsonpath='{.items[0].metadata.name}')
ncn-m001:~ # kubectl exec -n user $SLURMCTLD_POD -c slurmctld -- scontrol reconfigure
```

After the configuration job has been restarted, wait about 30 seconds to see the new content appear inside the pod.

If you were only adding or modifying burst_buffer.lua, then Slurm is now ready for your jobs.  If you were also modifying slurm.conf to set the BurstBufferType, or you were adding or modifying burst_buffer.conf, then you must also restart the Slurm pod.

```bash
ncn-m001:~ # kubectl rollout restart deployment -n user slurmctld
```

## Submit a test job

Use a UAN host to submit a test job.  The following assumes a UAN host named `uan01` and a user account named `vers`.

```bash
ncn-m001:~ # ssh uan01
uan01:~ # su vers
vers@uan01:/root> cd /lus/vers
vers@uan01:/lus/vers> cat > /tmp/dws-test <<EOF
#!/bin/sh                
#SBATCH --time=1
#DW
/bin/hostname
srun -l /bin/hostname
srun -l /bin/pwd
EOF
```

Note that we will submit the test job from a directory where our `vers` account can write its output files.

```bash
vers@uan01:/lus/vers> sbatch /tmp/dws-test
Submitted batch job 7773
vers@uan01:/lus/vers> sacct | grep 7773
vers@uan01:/lus/vers> cat slurm-7773.out
```

Get the status of the workflow.

```bash
vers@uan01:/lus/vers> scontrol show bbstat workflow 7773 && echo
desiredState=DataIn currentState=DataIn status=Completed
```

Note that the workflow resource will no longer exist after Slurm has completed its teardown state for this job.

## Inspect DWS Workflow resources

Inspect DWS Workflow resources by using the fully-qualified CRD name.

```bash
ncn-m001:~ # kubectl get workflows.dws.cray.hpe.com -A
```

<details>
<summary>Multiple CRDs named "Workflow"</summary>
    The fully-qualified name will protect you from picking up any resource named "Workflow" that belongs to some other service.

Your system may have multiple CRDs named "Workflow".

```bash
ncn-m001:~ # kubectl get crds | grep -E '^workflows\.'
workflows.argoproj.io                            2022-11-25T02:56:48Z
workflows.dws.cray.hpe.com                       2022-11-21T15:38:13Z
```
</details>

