---
authors: Matt Richerson <matthew.richerson@@hpe.com>
categories: operation
---

# Workflow Errors

A workflow may fail to complete the desired state for many reasons. The best action for the WLM to take depends on what is preventing the workflow from progressing. DWS communicates error information back to the WLM through a few different fields so the WLM can take the best course of action.

## Drivers and Directives

Most of the work required to move a workflow into the desired state is performed by software outside of DWS. These external pieces of software are called drivers. An example of a driver is the NNF software.

Directives are the #DW lines an end user provides as part of their job. There can be more than one directive in a job, and therefore more than one directive in a DWS workflow.

Directives are processed individually by the underlying drivers. At workflow creation time, the drivers modify the workflow to indicate which workflow states they are responsible for completing. This occurs for each of the directives. For each workflow state, DWS then waits for all the drivers to complete their work for each of the directives before marking the entire workflow as complete.

## Workflow Status

The current status of the workflow is reported in the `workflow.status.status` field. The following summarizes the values for this field:

1. `DriverWait` - The workflow is progressing towards completion, but it's waiting on one or more drivers to complete their work
2. `Completed` - All drivers have finished their work and the workflow has reached the state in `workflow.status.state`
3. `TransientCondition` - One or more drivers have encountered an error that might resolve
4. `Error` - One or more drivers have encountered an error that will not recover

If there are multiple directives in the workflow, the directive with the highest severity status is reported in the workflow.

## Workflow Message

The `workflow.status.message` field provides a human readable message about the current progress of the workflow. When the `workflow.status.status` is `TransientCondition` or `Error`, the `message` field is an error message. This error message is meant to provide useful information about the error to end users.

The error messages are prefixed with the index of the directive in the workflow that generated the error. For example `DW Directive 2`.

These error messages are also prefixed with one of three labels:

1. `User error` - A driver was unable to progress due to a bad input from an admin or an end user
2. `WLM error` - A driver was unable to progress due to a bad input from the WLM
3. `Internal error` - A driver was unable to progress due to an unexpected software or hardware issue

Below is an example of a full error message:
```
DW Directive 0: User error: only a single create_persistent or destroy_persistent directive is allowed per workflow
```

If there are multiple directives that encountered an error of the same severity, only one of the error messages is reported back up to the `workflow.status.message` field.

Depending on the type of error, the `workflow.status.message` field may not provide enough information for an admin to determine exactly what went wrong. This field is meant for end users, so the message purposefully hides driver implementation details.

## Internal Error Message

A more thorough debug message can be found in the `workflow.status.drivers[].message` field. The `workflow.status.drivers` array has an entry for each of the workflow states that a driver has registered for a directive. By using the directive index from the `workflow.status.message` field and the current workflow state, the correct array entry can be found. The `workflow.status.drivers[].dwdIndex` and `workflow.status.drivers[].watchState` fields contain the index and workflow state.

The error message in the `workflow.status.drivers[].message` will contain more driver specific detail about the error. This message is not intended for end users, but it may be useful for a system admin. An example error message is as follows:

```
internal error: storage resource error: default/lustre-mgs-pool-0: unable to format file system for allocation 0: could not create file share: Error 500: Internal Server Error, Retry-Delay: 0s, Cause: File share 'default-lustre-mgs-pool-0-mgt-0-0' failed to create, Internal Error: Error 500: Internal Server Error, Retry-Delay: 0s, Resource: #FileShare.v1_2_0.FileShare, Cause: File share 'default-lustre-mgs-pool-0-mgt-0-0' create failed Internal Error: Error Running Command 'zpool create -O canmount=off -o cachefile=none zf2916ba-mgtpool-0 /dev/nvme12n1 /dev/nvme11n1 /dev/nvme10n1 /dev/nvme1n1 /dev/nvme4n1 /dev/nvme14n1 /dev/nvme16n1 /dev/nvme5n1 /dev/nvme8n1 /dev/nvme3n1 /dev/nvme2n1 /dev/nvme13n1 /dev/nvme15n1 /dev/nvme7n1 /dev/nvme6n1 /dev/nvme9n1', StdErr: cannot label 'nvme12n1': try using parted(8) and then provide a specific slice: -4
```

## WLM Actions

When a workflow fails to progress due to an error, the WLM should change its response based on the type of error. When `workflow.status.status` is `Error`, the workflow will not recover from the error, and the WLM should move the workflow to state `Teardown` immediately. When `workflow.status.status` is `TransientCondition`, the driver code will continue to retry in the background, and the issue may resolve. The WLM should wait a short amount of time before moving the workflow to state `Teardown`. Thirty seconds is the recommended wait period.