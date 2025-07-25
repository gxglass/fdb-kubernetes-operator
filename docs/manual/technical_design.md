# Technical Design

## Overview

This document aims to provide more technical details about how the operator works to help people who are using the operator to understand its operations and debug problems that they experience.

The operator is built using [Kubebuilder](https://book.kubebuilder.io), and this document will refer to concepts from Kubebuilder as well as the [Kubernetes core](https://kubernetes.io/docs/home/).

There are two main pieces to the operator: the **Custom Resource Definition** and the **Controller**. The Custom Resource Definition provides a schema for defining objects that represent a FoundationDB cluster. Users of the operator create Custom Resources using this schema. The Controller watches for events on these Custom Resources and performs operations to reconcile the running state with the desired state that is expressed in that resource.

Our operator currently uses the following custom resource definitions:

* [FoundationDBCluster](../cluster_spec.md)
* [FoundationDBBackup](../backup_spec.md)
* [FoundationDBRestore](../restore_spec.md)

The documents linked above contain the full specification of these resource definitions, so they may be a useful reference for the fields that we refer to in this document.

All of these resources are managed by a single controller, with a single deployment. Within the controller, we have separate reconciliation logic for each resource type.

When we use the term "cluster" in this document with no other qualifiers, we are referring to a FoundationDB cluster. We will refer to a Kubernetes Cluster as a "KC" for brevity, and to avoid overloading the word "cluster".

When we use the term "cluster status" in this document, it refers to the status of the `FoundationDBCluster` resource in Kubernetes. When we use the term "database status" in this document, it refers to the output of the `status json` command in `fdbcli`.

When we reference something that contains the name of the cluster, we will assume the cluster is named `sample-cluster`.

This document also assumes that you are familiar with the earlier content in the user manual. We especially recommend reading through the section on [Resources Managed by the Operator](resources.md), which describes terminology and concepts that are used heavily in this document.

## Reconciliation Loops

The operations of our controller are structured as a reconciliation loop. At a high level the reconciliation loop works as follows:

1. The controller receives an update to the spec for a custom resource.
1. The controller identifies what needs to change in the running state to match that spec.
1. The controller makes whatever changes it can to converge toward the desired state.
1. If the controller encounters an error or needs to wait, it requeues the reconciliation and tries again later.
1. Once the running state matches the desired state, the controller marks the reconciliation as complete.

There are important constraints on how reconciliation has to work within this model:

* We have to be able to start reconciliation over from the beginning, which means changes must be idempotent.
* All Kubernetes resources are fetched through a local cache, which means all reads are potentially stale.
* Any local state that is not saved in a Kubernetes resource or the database may be lost at any time.
* We cannot compare the new spec to the previous spec to know what has changed. We can only compare the new spec to the live state, or to the information we store in the resource status or in other resources that we create.

In our operator, we add an additional abstraction to help structure the reconciliation loop, which we call a **Subreconciler**.
A subreconciler represents a self-contained chunk of work that brings the running state closer to the spec.
Each subreconciler receives the latest custom resource, and is responsible for determining what actions if any need to be run for the activity in its scope.
We run every subreconciler for every reconciliation, with the subreconcilers taking care of logic to exit early if they do not have any work to do.

Per default, since `v1.19.0`, the operator will fetch the [machine-readable status](https://apple.github.io/foundationdb/mr-status.html) at the start of a reconciliation loop and pass down the parsed status to all subreconcilers.
This reduces the need for fetching the `machine-readable status` multiple times in a single reconcile loop, for large clusters this has a significant performance improvement.
The risk for using the same `machine-readable status` for a single reconciliation loop is minimal, as a reconciliation loop normal takes only a few milliseconds to seconds.
Users can deactivate the caching per reconciliation loop by passing `--cache-database-status=false` as an argument to the operator.

## Locking Operations

This document will note which operations require a lock in order to complete.
This means that the operator needs to ensure that it is the only instance of the operator acting on the cluster, to prevent conflicts in multi-DC clusters.
For more using and configuring on the locking system, see the section on [Coordinating Global Operations](fault_domains.md#coordinating-global-operations).

The locking system works by setting a key in the database to indicate which instance of the operator can perform global operations. This key is `\xff\x02/org.foundationdb.kubernetes-operator/global`. This key will be set to a value of `tuple.Tuple{lockID,start,end}`. `lockID` is the `processGroupIDPrefix` from the cluster spec. `start` is a 64-bit integer representing a Unix timestamp with precision to the second, giving the time when this instance of the operator took the lock. `end` is a similar timestamp representing the time when the lock will automatically expire. The default lock duration is 10 minutes. If the operator tries to acquire a lock and sees that it already has the lock, it will extend it for another 10 minutes past the current time. If it sees that another instance of the operator has a lock, and the current time is past the end of the lock, it will clear the old lock and take a new lock for itself. If it sees that another instance of the operator has a lock, and the current time is before the end of the lock, it will requeue reconciliation until it can acquire the lock.

The locking system is used to protect operations that have global scope or otherwise have a global impact. This includes operations like setting database configuration, which impacts the entire cluster. It also includes operations that trigger recoveries or that we want to restrict to one DC at a time, such as excluding processes.

Because this locking system involves writing to the database, it will not work when the database is unavailable. In that situation any attempt to aquire a lock will fail. If the database is unavailable and you need the operator to take action to make it available, you can work around this by setting the `disableLocks` field in the lock options to `true`. However, many of the actions that require locks are activities that are impossible or unsafe when the database is unavailable, and often an unavailable database will require manual intervention.

If there is a dysfunctional instance of the operator that cannot be trusted to perform global operations, you can block it from taking locks by adding its `lockID` to the deny list in the cluster spec. You can set this value in any DC. This will only affect operations on the cluster whose spec you update. This will set the key `\xff\x02/org.foundationdb.kubernetes-operator/denyList/$lockID` to the the value `$lockID`. If an instance of the operator with that lock ID sees that the key is set, it will fail any attempt to acquire a lock, even if it has a lock already. Any other instance of the operator that sees an active lock for an instance in the deny list will ignore that lock and will be able to take one for itself.

In order to avoid contention between different instances of the operator in managing the deny list, if the operator has no entry in the deny list in its spec for a given `lockID`, it will take no action. This means that if you set the deny list in the spec, and then clear that field in the spec, the deny list will still be set in the database. In order to effectively remove an entry from the deny list, you have to update its entry with the flag `allow: true`. This tells the operator that your intention is to explicitly allow this instance to take global locks again.

As an example, you would use this spec to add the operator in `dc1` to the deny list:

```yaml
spec:
    lockOptions:
        denyList:
            - id: dc1
```

And you would use this spec to remove the operator in `dc1` from the deny list:

```yaml
spec:
    lockOptions:
        denyList:
            - id: dc1
              allow: true
```

Once that is done and the change is reconciled, you can clear the deny list in the spec.

See the [LockOptions](../cluster_spec.md#LockOptions) documentation for more options for customizing the locking system.

## Cluster Reconciliation

The cluster reconciler runs the following subreconcilers:

1. [UpdateStatus](#updatestatus)
1. [UpdateLockConfiguration](#updatelockconfiguration)
1. [UpdateConfigMap](#updateconfigmap)
1. [CheckClientCompatibility](#checkclientcompatibility)
1. [DeletePodsForBuggification](#deletepodsforbuggification)
1. [ReplaceMisconfiguredProcessGroups](#replacemisconfiguredprocessgroups)
1. [ReplaceFailedProcessGroups](#replacefailedprocessGroups)
1. [AddProcessGroups](#addprocessgroups)
1. [AddServices](#addservices)
1. [AddPVCs](#addpvcs)
1. [AddPods](#addpods)
1. [GenerateInitialClusterFile](#generateinitialclusterFile)
1. [RemoveIncompatibleProcesses](#removeincompatibleprocesses)
1. [UpdateSidecarVersions](#updatesidecarversions)
1. [UpdatePodConfig](#updatepodconfig)
1. [UpdateLabels](#updatelabels)
1. [UpdateDatabaseConfiguration](#updatedatabaseconfiguration)
1. [ChooseRemovals](#chooseremovals)
1. [ExcludeProcesses](#excludeprocesses)
1. [ChangeCoordinators](#changecoordinators)
1. [BounceProcesses](#bounceprocesses)
1. [UpdatePods](#updatepods)
1. [RemoveProcessGroups](#removeprocessgroups)
1. [RemoveServices](#removeservices)
1. [UpdateStatus (again)](#updatestatus)

### Tracking Reconciliation Stages

We track the progress of reconciliation through a `Generations` object, in the `status.generations` field in the cluster object. The generation status has fields within it that indicate how far reconciliation has gotten, with an integer for each field indicating the generation that was seen for that reconciliation. The most important field to track here is the `reconciled` field, which is set when we consider reconciliation _mostly_ complete. If you want to track a rollout, you can check for whether the generation number in `status.generations.reconciled` is equal to the generation number in `metadata.generation`.

There are some cases where we set the `reconciled` field to the current generation even though we are requeuing reconciliation and continuing to do more work. These cases are listed below:

1. Pods are in terminating. If we have fully excluded processes and have started the termination of the pods, we set both `reconciled` and `hasPendingRemoval` to the current generation. Termination cannot complete until the kubelet confirms the processes has been shut down, which can take an arbitrary long period of time if the kubelet is in a broken state. The processes will remain excluded until the termination completes, at which point the operator will include the processes again and the `hasPendingRemoval` field will be cleared. In general it should be fine for the cluster to stay in this state indefinitely, and you can continue to make other changes to the cluster. However, you may encounter issues with the stuck pods taking up resource quota until they are fully terminated.

### UpdateStatus

The `UpdateStatus` subreconciler is responsible for updating the `status` field on the cluster to reflect the running state. This is used to give early feedback of what needs to change to fulfill the latest generation and to front-load analysis that can be used in later stages. We run this twice in the reconciliation loop, at the very beginning and the very end. The `UpdateStatus` subreconciler is responsible for updating the generation status and the ProcessGroup conditions.

### UpdateLockConfiguration

The `UpdateLockConfiguration` subreconciler sets fields in the database to manage the deny list for the cluster locking system. See the [Locking Operations](#locking-operations) section for more information about this locking system.

### UpdateConfigMap

The `UpdateConfigMap` subreconciler creates a `ConfigMap` object for the cluster's configuration, and updates it as necessary. It is responsible for updating the labels and annotations on the `ConfigMap` in addition to the data.

### CheckClientCompatibility

The `CheckClientCompatibility` subreconciler is used during upgrades to ensure that every client is compatible with the new version of FoundationDB. When it detects that the `version` in the cluster spec is protocol-compatible with the `runningVersion` in the cluster status, this will do nothing. When these are different, it means there is a pending upgrade. This subreconciler will check the `connected_clients` field in the database status, and if it finds any clients whose max supported protocol version is not the same as the `version` from the cluster spec, it will fail reconciliation. This prevents upgrading a database until all clients have been updated with a compatible client library.

This subreconciler will also ensure that if the versions are protocol-incompatible, the new version is more recent than the cluster's current version. The level of support for protocol-incompatible downgrades is more nuanced than the path for upgrades, and the [FoundationDB docs](https://apple.github.io/foundationdb/administration.html#version-specific-notes-on-downgrading) have more details on this path.

You can skip these checks by setting the `ignoreUpgradabilityChecks` flag in the cluster spec.

### DeletePodsForBuggification

The `DeletePodsForBuggification` subreconciler deletes pods that need to be recreated in order to set buggification options. These options are set through the `buggify` section in the cluster spec.

When pods are deleted for buggification, we apply fewer safety checks, and buggification will often put the cluster in an unhealthy state.

### ReplaceMisconfiguredProcessGroups

The `ReplaceMisconfiguredProcessGroups` subreconciler checks for process groups that need to be replaced in order to safely bring them up on a new configuration. The core action this subreconciler takes is setting the `removalTimestamp` field on the `ProcessGroup` in the cluster status. Later subreconcilers will do the work for handling the replacement, whether processes are marked for replacement through this subreconciler or another mechanism.

See the [Replacements and Deletions](replacements_and_deletions.md) document for more details on when we do these replacements.

### ReplaceFailedProcessGroups

The `ReplaceFailedProcessGroups` subreconciler checks for process groups that need to be replaced because they are in an unhealthy state. This only takes action when automatic replacements are enabled. The core action this subreconciler takes is setting the `removalTimestamp` field on the `ProcessGroup` in the cluster status. Later subreconcilers will do the work for handling the replacement, whether processes are marked for replacement through this subreconciler or another mechanism.

See the [Replacements and Deletions](replacements_and_deletions.md) document for more details on when we do these replacements.

### AddProcessGroups

The `AddProcessGroups` subreconciler compares the desired process counts, calculated from the cluster spec, with the number of process groups in the cluster status. If the spec requires any additional process groups, this step will add them to the status. It will not create resources, and will mark the new process groups with conditions that indicate they are missing resources.

### AddServices

The `AddServices` subreconciler creates any services that are required for the cluster. By default, the operator does not create any services. If the `routing.headless` flag in the spec is set, we will create a headless service with the same name as the cluster. If the `routing.publicIPSource` field is set to `service`, we will create a service for every process group, with the same name as the pod.

### AddPVCs

The `AddPVCs` subreconciler creates any PVCs that are required for the cluster. A PVC will be created if a process group has a stateful process class, has no existing PVC, and has not been flagged for removal.

### AddPods

The `AddPods` subreconciler creates any pods that are required for the cluster. Every process group will have one pod created for it. If a process group is flagged for removal and a previous run of `RemoveProcessGroups` has determined (by submitting the `exclude` command to FoundationDB) that it has in fact been fully excluded from the FoundationDB cluster, we will not create a pod for it. However, if we do not know for certain that the process group is fully excluded from FoundationDB, we will bring it back up even if it is flagged for removal - this is to handle a case where a storage node crashes (or is accidentally stopped) while it is draining.

### GenerateInitialClusterFile

The `GenerateInitialClusterFile` creates the cluster file for the cluster. If the cluster already has a cluster file, this will take no action. The cluster file is the service discovery mechanism for the cluster. It includes addresses for coordinator processes, which are chosen statically. The coordinators are used to elect the cluster controller and inform servers and clients about which process is serving as cluster controller. The cluster file is stored in the `connectionString` field in the cluster status. You can manually specify the cluster file in the `seedConnectionString` field in the cluster spec. If both of these are blank, the operator will choose coordinators that satisfy the cluster's fault tolerance requirements. Coordinators cannot be chosen until the pods have been created and the processes have been assigned IP addresses, which by default comes from the pod's IP. Once the initial cluster file has been generated, we store it in the cluster status and requeue reconciliation so we can update the config map with the new cluster file.

### RemoveIncompatibleProcesses

The `RemoveIncompatibleProcesses` subreconciler will check the FoundationDB cluster status for incompatible connections.
If the cluster has some incompatible connections the subreconciler will match those IP addresses with the process groups.
For matching process groups the subrecociler will delete the associated Pod and let it recreate with the new image.

### UpdateSidecarVersions

The `UpdateSidecarVersions` subreconciler updates the image for the `foundationdb-kubernetes-sidecar` container in each pod to match the `version` in the cluster spec.
Once the sidecar container is upgraded to a version that is different from the main container version, it will copy the `fdbserver` binary from its own image to the volume it shares with the main container, and will rewrite the monitor conf file to direct `fdbmonitor` to start an `fdbserver` process using the binary in that shared volume rather than the binary from the image used to start the main container.
This is done temporarily in order to enable a simultaneous cluster-wide upgrade of the `fdbserver` processes.
Once that upgrade is complete, we will update the image of the main container through a rolling bounce, and the newly updated main container will use the binary that is provided by its own image.

### UpdatePodConfig

The `UpdatePodConfig` subreconciler synchronizes updates to the config map with a pod's local state. When the kubelet detects an update to the config map, it updates the local contents in the sidecar container, through the input-files mount. The sidecar is responsible for copying the files into its output-files mount, which is shared with the main container. For some files, such as the cluster file, the sidecar directly copies the file. For the monitor conf file, the sidecar provides some template substitution to replace placeholder strings in the monitor conf with values supplied through environment variables. This substitution allows us to use a single monitor conf file for multiple pods, with pod-specific values like the node name supplied dynamically. This copying process is triggered by the operator through the sidecar's API. The operator also uses this API to verify the hashes of the files, confirming that the pod has the latest configuration. Once this is confirmed, the operator updates the pod with an annotation containing a hash of the config map contents. If the current hash in the annotations matches the desired contents, the operator takes no actions on the pod.

This process can only succeed if several things are true:

* The sidecar container must be running and healthy
* The FDB pod being updated must be reachable from the operator pod
* The kubelet must be connected to the API server so that it can fetch the latest config map information

If these things are not true, then the operator will requeue reconciliation. This can cause reconciliation to get blocked indefinitely when a pod is unhealthy. To work around this, you can tell the operator to replace the pod. If a pod is flagged for removal, then the operator will not try to update its config in this action.

### UpdateLabels

The `UpdateLabels` subreconciler updates the labels and annotations for the resources created by the operator based on the process settings, as well as setting core labels and annotations that the operator uses for its own purposes. Any labels or annotations that do not have values specified in the spec will be left unmodified. This means that if you define a label in the cluster spec, and then remove that label from the spec, you will have to manually remove it from any existing resources in order for the label to completely go away.

### UpdateDatabaseConfiguration

The `UpdateDatabaseConfiguration` subreconciler runs `configure` commands in `fdbcli` to ensure that the active database configuration matches the configuration in the cluster spec. In most cases, this will mean running a single `configure` command. However, there are some configuration changes that have to be done in multiple stages with time between them for the database to stabilize and replicate data. Changes to region configuration in multi-DC clusters are an example of this multi-stage configuration. The operator will automatically break up these configuration changes into batches that the database can process, and will requeue reconciliation after making each change until it reaches the full desired configuration.

The operator uses the `configured` field in the cluster status to determine if it is needs to do the initial database configuration, which means running a `configure new` command. As soon as the operator detects that the database has a database configuration, or performs a database configuration itself, it will set the `configured` field to `true`. After that point it will never run a `configure new` command.

If the database is unavailable, the operator will not attempt any configuration changes, but will move forward with reconciliation in case a later stage can restore the database availability. If the database is available but has unhealthy data distribution, the operator will move forward with reconciliation. As part of the `UpdateStatus` subreconciler, the operator will compare the live database configuration against the spec and will not consider reconciliation complete until the live configuration is up-to-date.

This action requires a lock.

### ChooseRemovals

The `ChooseRemovals` subreconciler flags processes for removal when the current process count is more than the desired process count. The processes that are removed will be chosen so that the remaining process are spread across as many fault domains as possible. The core action this subreconciler takes is setting the `removalTimestamp` field on the `ProcessGroup` in the cluster status. Later subreconcilers will do the work for handling the removal.

### ExcludeProcesses

The `ExcludeProcesses` subreconciler runs an [exclude command](https://apple.github.io/foundationdb/administration.html#removing-machines-from-a-cluster) in `fdbcli` for any process group that is marked for removal and is not already being excluded.
The `exclude` command tells FoundationDB that a process should not serve any roles, and that any data on that process should be moved to other processes.
This exclusion can take a long time, but this subreconciler does not wait for exclusion to complete.
The operator will only run the exclude command if it is safe to run it.
The safety checks are defined in the [status_checks.go](../../pkg/fdbstatus/status_checks.go) file and includes the following checks:

- There is a low number of active generations.
- The cluster is available from the client perspective.
- The last recovery was at least `MinimumRecoveryTimeForExclusion` seconds ago.

The `MinimumRecoveryTimeForExclusion` parameter can be changed with the `--minimum-recovery-time-for-exclusion` argument and the default is `120.0` seconds.
Having a wait time between the exclusions will reduce the risk of successive recoveries which might cause issues to clients.

The operator will only trigger a replacement if the new processes are available.
In addition the operator will not trigger any exclusion if any of the process groups with the same process clas has the `MissingProcess` condition for less than 5 minutes.
This reduces the risk of multiple exclusions, and recoveries, during a migration.
If a process group has the `MissingProcess` condition for more than 5 minutes it will be ignored and the exclusions might proceed.
This mechanism reduces the risk that a migration gets stuck because of resource quota limitations.

The operator will calculate the "budget" of processes that can be excluded on a process class basis.
The calculation takes the desired process count, ongoing exclusions and missing processes into account:

```go
// All processes without the MissingProcess condition are considered valid in this case.
len(validProcesses) - desiredProcessCount - ongoingExclusions
```

If the budget is greater than 0 the operator will exclude as many processes as the budget allows.
If the budget is 0 or less, the operator will wait for new processes to come up.

In most cases this will allow the operator to move forward with the exclusions and the migration, even if the resources are limited.
There are some cases that could get the operator still stuck, e.g. if not enough new Pods can be created to allow the operator to choose new coordinators.

In this case you can unblock the operator by either increasing the quota of the namespace during the migration or you could manually exclude some processes with `fdbcli`.
If you decide to manually exclude processes, you should make sure that the replication factor can still be satisfied.

### ChangeCoordinators

The `ChangeCoordinators` subreconciler ensures that the cluster has a healthy set of coordinators that fulfill the fault tolerance requirements for the cluster. If any coordinators have failed, or if the database configuration requires more coordinators or better-distributed coordinators, the operator will choose new coordinators and run a `coordinators` command to tell the database to use the new set. It will then read the new connection string and update it in the cluster status.

This will recruit coordinators based on the process list in the database status to ensure that the coordinators it recruits are properly connecting to the database. This will prefer to recruit coordinators only from `storage` processes. If it cannot fulfill the fault tolerance requirements using storage requirements, it will expand the candidate list to include `log` processes, and then to include `transaction` processes if necessary. It will ensure that the coordinators are distributed across failure domains as evenly as possible. It will also require that every coordinator has a different `zoneid` locality. For multi-DC clusters, it will require that we do not have a majority of coordinators using the same value for the `dcid` locality.

For single-DC clusters, the number of coordinators will be `2R-1`, where `R` is the replication factor. For multi-DC clusters, we will always use 9 coordinators.

This action requires a lock.

### BounceProcesses

The `BounceProcesses` subreconciler restarts any `fdbserver` processes that do not have the correct command line. This is done through the `kill` command in fdbcli, which causes the processes to immediately exit, which causes `fdbmonitor` to restart them. This will restart any process for a process group that has the `IncorrectCommandLine` condition.

When upgrading a cluster to a new version of FoundationDB, we follow a special process. In most cases, each instance of the operator only restarts processes that are under its control, which means that in multi-KC clusters we will restart processes in multiple batches, with one batch for each KC. During an upgrade, we cannot use this strategy, because protocol-incompatible upgrades require all processes to be updated simultaneously. To make this work, we have each instance of the operator use the locking system to store a list of processes that it has prepared for the upgrade in the database. Each instance of the operator then checks that list and compares it against the database status to confirm that every process that is reporting to the database is ready for the upgrade. It will then restart all of the processes across the entire cluster and move forward with its own reconciliation. When the other instances of the operator run their next reconciliation, they will see that the processes they are managing have the correct command-line, and will move past the bounce stage.

If a process needs to be restarted but is not reporting to the database, this will requeue reconciliation with an error.

This will not attempt to restart any process that is flagged for removal.

This will not restart processes until every process has been up for 600 seconds. This limit can be configured through the `minimumUptimeSecondsForBounce` field in the cluster spec.

This action requires a lock.

### UpdatePods

The `UpdatePods` subreconciler deletes any pods that have incorrect pod specs. Once it deletes a pod, it will requeue reconciliation so that the operator can recreate the pod on the next reconciliation run.

This will only delete pods with a single `zoneid` locality value, which ensures that we only lose one unit of fault tolerance through these restarts.

This will not delete any pods that are flagged for removal.

If any pod is in a terminating state and is not flagged for removal, this will not delete any further pods. It will requeue reconciliation until the in-flight termination completes.

This action requires a lock.

### RemoveServices

The `RemoveServices` subreconciler deletes any services that are no longer required for the cluster.

### RemoveProcessGroups

The `RemoveProcessGroups` subreconciler deletes any pods that are marked for removal and have been fully excluded, meaning that they are not serving any roles or holding any data.

This performs the following sequence of steps for every pod:

1. Confirm that the exclusion is complete.
1. Trigger the deletion of the pod.
1. Confirm that the pod is fully terminated.
1. Trigger the deletion of the PVC.
1. Confirm that the PVC is fully terminated.
1. Trigger the deletion of the service.
1. Confirm that the service is fully terminated.
1. Include the processes where all resources are deleted.
1. Remove the process group from the cluster's list of process groups.

If any process group is marked for removal but cannot complete the sequence above, this will requeue reconciliation.
However, we will always run through this sequence on all of the process groups that need to be removed, getting as far as we can for each one.
This means that one pod being stuck in terminating should not block other pods from being deleted.

This will not allow deleting any pods that are serving as coordinators.

The `RemoveProcessGroups` subreconciler has some additional safety checks to reduce the risk of successive recoveries.

The operator will only run the exclude command if it is safe to run it.
The safety checks are defined in the [status_checks.go](../../pkg/fdbstatus/status_checks.go) file and includes the following checks:

- There is a low number of active generations.
- The cluster is available from the client perspective.
- The last recovery was at least `MinimumRecoveryTimeForExclusion` seconds ago.

The `MinimumRecoveryTimeForExclusion` parameter can be changed with the `--minimum-recovery-time-for-exclusion` argument and the default is `120.0` seconds.
Having a wait time between the exclusions will reduce the risk of successive recoveries which might cause issues to clients.

The same is true for the include operation with the difference that `MinimumRecoveryTimeForInclusion` is used to determine the minimum uptime of the cluster.
The `MinimumRecoveryTimeForInclusion` parameter can be changed with the `--minimum-recovery-time-for-inclusion` argument and the default is `600.0` seconds. 
The operator will batch all outstanding inclusion together into a single include call.

### UpdateStatus (again)

Once we have completed all other steps in reconciliation, we run the `UpdateStatus` subreconciler a second time to check that everything is in the desired state. If there is anything that is not in the desired state, the operator will requeue reconciliation.

## Backup Reconciliation

The backup reconciler runs the following subreconcilers:

1. UpdateBackupStatus
1. UpdateBackupAgents
1. StartBackup
1. StopBackup
1. ToggleBackupPaused
1. ModifyBackup
1. UpdateBackupStatus (again)

### UpdateBackupStatus

The `UpdateBackupStatus` subreconciler is responsible for updating the `status` field on the backup to reflect the running state. This is used to give early feedback of what needs to change to fulfill the latest generation and to front-load analysis that can be used in later stages. We run this twice in the reconciliation loop, at the very beginning and the very end. The `UpdateBackupStatus` subreconciler is responsible for updating the generation status.

### UpdateBackupAgents

The `UpdateBackupAgents` subreconciler is responsible for creating and updating the deployment for running the `backup_agent` processes.

### StartBackup

The `StartBackup` subreconciler is responsible for starting a backup. If a backup is supposed to be running, but the database status reports no ongoing backup, this will run the `start` command in `fdbbackup`.

### StopBackup

The `StopBackup` subreconciler is responsible for stopping a backup. If a backup is not supposed to be running, but the database status reports an ongoing backup, this will run the `stop` command in `fdbbackup`.

### ToggleBackupPaused

The `ToggleBackupPaused` subreconciler is responsible for pausing and unpausing a backup. Pausing a backup means that the backup will be configured in the cluster, but the backup agents will not do any work. If the desired state of the backup is `Paused`, and the backup is not paused, this will run the `pause` command in `fdbbackup`.  If the desired state of the backup is `Running`, and the backup is paused, this will run the `resume` command in `fdbbackup`.

### ModifyBackup

The `ModifyBackup` command ensures that any properties that can be configured on a live backup are configured to the values in the backup spec. This will run the `modify` command in `fdbbackup` to set the properties from the spec.

Currently, this only supports the `snapshotPeriodSeconds` property.

### UpdateBackupStatus (again)

Once we have completed all other steps in reconciliation, we run the `UpdateBackupStatus` subreconciler a second time to check that everything is in the desired state. If there is anything that is not in the desired state, the operator will requeue reconciliation.

## Restore Reconciliation

The restore reconciler runs the following subreconcilers:

1. StartRestore

### StartRestore

The `StartRestore` subreconciler starts a restore. If there is no active restore, this will run the `start` command in `fdbrestore`.

## Interaction Between the Operator and the Pods

The operator communicates with processes running inside the FoundationDB pods at multiple stages in the reconciliation flow. The exact flow will depend on whether you are using split images (which is the current default) or unified images.

### Split Image

When using split images, the `foundationdb` container runs a `fdbmonitor` process, which is responsible for starting the `fdbserver` processes. `fdbmonitor` receives its configuration in the form of a monitor conf file, which contains the start command and arguments for the fdbserver process. This configuration can vary based on dynamic information like the node where the pod is running, so the operator provides a templated configuration file, which contains placeholders that are filled in based on the environment variables. That templating process is handled by the sidecar process in the `foundationdb-kubernetes-sidecar` container. The sidecar also provides an HTTP API for getting information about the state of the configuration in the pod. The sidecar mounts the config map containing dynamic conf as its input directory, and shares an `emptyDir` volume with the `foundationdb` container where it can put its output.

The flow for updating the fdbserver processes has the following steps:

1. The operator updates the monitor conf template in the `sample-cluster-config` config map. There is one monitor conf template for every process class the cluster uses.
2. The operator calls the sidecar API to check the hash of the output monitor conf and compare the hash to the desired contents.
3. The operator sees that the config is out of date, and sets an annotation on the pod to indicate that it needs an update.
4. Kubernetes fetches the contents of the config map from the API server and updates the template in the sidecar container.
5. The operator tells the sidecar to regenerate the monitor conf based on the new template. The sidecar places the generated monitor conf in its output directory.
6. The operator checks the latest output monitor conf to see if it is correct.
7. Once the monitor conf is correct, the operator uses the CLI to shut down the fdbserver processes.
8. fdbmonitor sees that the processes have exited and starts new processes with the latest configuration.

The operator follows a similar process when the `fdb.cluster` file needs to be updated. However, because this file is not templated, the sidecar simply copies the file from the input directory to the output directory. Cluster file updates do not require restarting processes.

When the operator checks the status of the cluster, it needs to check if the process start commands are an exact match for the expected values based on the cluster spec. In order to make this comparison, it needs to fill in pod-specific information like the address and node name. The sidecar also provides an API for reading the environment variables that are being referenced in the monitor conf, and what their current values are. The operator uses this API when performing this check on the start command.

The sidecar has an important role to play in the upgrade flow. The monitor conf template uses a template variable `$BINARY_DIR` for the directory where the `foundationdb` container should look for the `fdbserver` binary. The sidecar process sets this template variable based on its understanding of the versions of the main container and the sidecar container. When they are running the same version of FDB, the `$BINARY_DIR` is set to the directory with the binaries that are provided by the `foundationdb` image. When they are running a different version, the sidecar copies the FDB binaries from its own image into the output directory, and sets the `$BINARY_DIR` to the path to these binaries in that directory.

### Unified Image

When using the unified image, the `foundationdb` container runs a `fdb-kubernetes-monitor` process, which is responsible for starting `fdbserver` processes.
`fdb-kubernetes-monitor` receives its configuration in the form of a JSON file, which provides the command-line arguments in a structured form.
These arguments can reference environment variables, which will be filled in by `fdb-kubernetes-monitor`.
They can also reference the process number, which allows `fdb-kubernetes-monitor` to start multiple `fdbserver` processes that use different ports and different data directories.

The flow for updating the monitor conf file has the following steps:

1. The operator updates the monitor conf template in the `sample-cluster-config` config map. There is one monitor conf file for every process class the cluster uses.
2. The operator checks the annotations on the pod to see the monitor conf that the pod is currently using.
3. The operator sees that the config is out of date, and sets an annotation on the pod to indicate that it needs an update.
4. Kubernetes fetches the contents of the config map from the API server and updates the template in the sidecar container.
5. fdb-kubernetes-monitor receives an event about the updated configuration file and loads it. It runs basic validations on the new conf.
6. If the new config is usable, fdb-kubernetes-monitor will store the new configuration as its active configuration and updates the annotations on the pod with the new configuration.
7. Once the operator sees that the active configuration matches the desired configuration, it uses the CLI to shut down the fdbserver processes.
8. fdb-kubernetes-monitor sees that that processes have exited and starts new processes with the latest configuration.

The active configuration is stored on the pod under the annotation `foundationdb.org/launcher-current-configuration`.

**NOTE**: Because the pod annotations are used to communicate the state in this flow, the pods must have a service account token that has permissions to read and write pods.

`fdb-kubernetes-monitor` does not watch the `fdb.cluster` for updates. Changes to the connection string will be sent directly to the fdbserver processes through the `coordinators` command in the CLI.

When the operator checks the status of the cluster, it needs to check if the process start commands are an exact match for the expected values based on the cluster spec. In order to make this comparison, it needs to fill in pod-specific information like the address and node name. fdb-kubernetes-monitor provides this information through the `foundationdb.org/launcher-environment` annotation on the pod, which contains a map of environment variables to their values. The operator uses this annotation when performing this check on the start command.

All of the flows above go through the `foundationdb` container. The `foundationdb-kubernetes-sidecar` container is only used in the upgrade flow. The sidecar container runs the same image as the main container, but with a different set of arguments to tell it to run in sidecar mode. During the upgrade, the operator upgrades the sidecar to the new version of FDB while leaving the main container at the old version. The sidecar compares the version of FoundationDB that it is running against the main container version, which is provided in its start command. If these versions are the same, the sidecar will do nothing. If they are different, it will copy the FDB binaries from its own image into a volume that it shares with the main container. The main container will receive the desired version of FDB as part of its configuration file. When the main container sees a version of FDB that is different from the one it is running, it will look for the FDB binaries in the directory it shares with the sidecar. If it finds those new binaries, it will load the new configuration and run the binaries from that directory. If these binaries are missing, fdb-kubernetes-monitor will reject the new configuration. Once the new configuration is accepted by all of the pods, the operator will restart the processes so they start running with the new binaries. Once the new version is running, the operator will perform a rolling bounce to update the main container to the new FDB version.

## Interaction Between the Operator and the FoundationDB cluster

The operator will use the [machine-readable status](https://apple.github.io/foundationdb/mr-status.html) of FoundationDB to observe the current state of the FoundationDB cluster.
Based on the machine-readable status the operator can issue commands to reconcile to the desired state.

### Exclusions in the operator

The operator uses the [exclude](https://apple.github.io/foundationdb/command-line-interface.html#exclude) command to make sure it's safe [to remove a Pod from the cluster](https://apple.github.io/foundationdb/administration.html#removing-machines-from-a-cluster).
The exclusion is handled in the [exclusion subreconciler](#excludeprocesses) and will make sure that at least a minimum number of processes are up and running before issuing the `exclude` command.
When the operator observes that some processes must be excluded it tries to exclude them all at once as the exclusion command can trigger a recovery in the FoundationDB cluster.
A process marked as `excluded` in the machine-readable status means that the FoundationDB cluster will not consider this specific process as an eligible process for the transaction system and all data will be moved away from this process.
The operator will also make sure to select new coordinators if at least one coordinator is marked as excluded, this is handled in the [change coordinators subreconciler](#changecoordinators).

In order to be able to verify if it's safe to remove the resources of a process the operator will check the roles of the excluded processes.
If a process is serving no roles and is marked as excluded, it's safe to remove the resources of this process.
If a process has at least one role, it's not safe to remove this process.
If a process is missing in the machine-readable status the operator will issue an additional `exclude` command for those missing processes to ensure they are not serving any log or storage roles.

The current default for the operator is to use the Pod IP for the exclusion command, if a Pod get's deleted and recreated it could get a new IP address and the operator has to issue a new exclude command for the new IP address.
To workaround this FoundationDB added support for locality based exclusions in 7.0 and the operator supports this by setting [useLocalitiesForExclusion](https://github.com/FoundationDB/fdb-kubernetes-operator/blob/main/docs/cluster_spec.md#foundationdbclusterautomationoptions) in the FoundationDBCluster spec.

NOTE: the operator is not able to use the `failed` option for exclusions.

## Next

You can continue on to the [next section](upgrades.md) or go back to the [table of contents](index.md).
