# Rollback

## Scenario

Configuration mistakes and buggy new versions are bound to happen. A fast and safe rollback process that allows recovery to a known good version or configuration will increase plant operator confidence considerably.

### Requirements

1. The rollback process should work on a cluster level, and should be able to bring back the cluster configuration to a known good state. The cluster configuration encompasses the whole state of the cluster, but here is scoped to only the state of the cluster as represented in the gitops repository. Determination of the state to which the cluster should be rolled-back will be done by the operator (i.e. it is not automatically determined through automation).

2. Rollback should complete reasonably fast, ideally no slower than a regular configuration change.

3. The rollback process should not have an impact on cluster management after the rollback has been completed, i.e. the cluster should be operatable as per usual after a rollback, without need for special considerations. In effect, rollback should just be another configuration change, instead of a special disaster recovery mode or similar.

## Background

At least two generic approaches exist for rollback:

1. Using versioning to create immutable artifacts which can be rolled back to
2. Having a mechanism that can restore the previous configuration state from some kind of persistent storage, like the git history

As the configuration of the cluster is mainly stored in the gitops repository, the scope of the rollback must encompass the gitops repository. The configurations outside of the gitops repository are outside of scope of this document, but should also be taken into consideration.

## Rolling back and handling global directories

The gitops repository consists of the clusters directory, that houses cluster-specific configuration, and other directories like the `/apps` and `/.defaults` directories that contain configurations that apply to multiple clusters (global configuration).

**In order to fulfill requirement 3, it is not possible to modify the global configuration when rolling back a single cluster, thus approach 1 should be used for those directories.** This means that the apps, .defaults and other global directories should be append-only, i.e. existing application versions are immutable and only the creation of new versions is permitted, and all other configurations, like configmaps in .defaults, must be versioned as well.

The clusters configuration in the cluster directory however can be rolled back in several manners. Two good solutions were identified and are discussed below.

## Solution 1: Creating new deployableApplicationVersions

Approach 1 can be used for the clusters directory as well: each change to deployableApplications can be versioned with a deployableApplicationVersion. Thus, it would be possible to create a new deployableApplicationVersion for every configuration change, make deployableApplicationVersions immutable and roll back by changing back to a previous deployableApplicationVersion.

### Practical implementation

#### When changing cluster configuration

1. Before each configuration change, call `POST /Clusters/{cluster}/applicationSets/{applicationSet}/deployableApplications/{deployableApplication}/deployableApplicationVersions` to create a new deployableApplicationVersion
2. Make any configuration changes to the newly created deployableApplicationVersion using the Config Service API's or via the gitops repository.
3. Deploy the new deployableApplicationVersion with `POST /Clusters/{cluster}/applicationSets/{applicationSet}/deployableApplications/{deployableApplication}/deployableApplicationVersions/{deployableApplicationVersionToDeploy}/deploy`.

#### Rollback

1. Determine the deployableApplicationVersion that was deployed during the known good state, for example by looking at the history of the `clusters/<cluster-name>/sets/<applicationSet-name>/<deployableApplication-name>/kustomization.yaml` file.
2. Call `POST /Clusters/{cluster}/applicationSets/{applicationSet}/deployableApplications/{deployableApplication}/deployableApplicationVersions/{deployableApplicationVersionToDeploy}/deploy` to deploy the known good deployableApplicationVersion

### Disadvantages

* This approach only works for configurations stored under the `clusters/<cluster-name>/sets/<applicationSet-name>/<deployableApplication-name>/<deployableApplicationVersion>` directory, so notably it can not be used to roll back the addition or removal of deployableApplications or applicationSets onto a cluster.
* The approach works only when new deployableApplicationVersions are created for each configuration state to which rolling back should be possible. In practice, two approaches can be taken: either deployableApplicationVersions are treated as strictly immutable and each configuration change requires the creation of a new deployableApplicationVersion, or the rule could be relaxed so that for a limited set of configurations that change at a high velocity the creation of new deployableApplicationVersions is not mandatory. If the latter approach is chosen, operators should check the excluded configurations manually.
* If multiple deployableApplicationVersions should be rolled back at the same time, an Approval should be used to group all changes together into a single transaction.
* The solution creates a large number of deployableApplicationVersions, which in turn creates a large number of files and directories on the gitops repository.

### Advantages

* This approach is already supported in Config Service and does not require any code changes to Config Service.

## Solution 2: rolling back via git history

Approach 2 can be used for the clusters directory: rollback can be done by changing the contents of the `clusters/<cluster-name>` directory to a known good state.

### Practical implementation

1. Determine the known good version of the `clusters/<cluster-name>` directory by determining which git commit was at HEAD of the gitops repository at the known good state. A more thorough approach would be to check which commit was reported as deployed onto the cluster at the known good state with observability tooling.
2. Fetch & check out the gitops repository to a working directory. Run
```bash
git checkout <healthy revision>
cp -r clusters/<device-to-rollback> /tmp/ #create copy of desired state
git checkout main
rm -rf clusters/<device-to-rollback> #create clean slate within local repository
cp -r /tmp/<device-to-rollback> clusters/ #get desired state into current iteration
rm -rf /tmp/<device-to-rollback> #clean up
git add clusters/<device-to-rollback>
git commit -m <message>

```

### Disadvantages

* The approach is not implemented in Config Service. New code in the Config Service or elsewhere must be written.
* Unless the rollback feature is implemented in the Config Service itself (as opposed to a shell script developers use or added into some other tool), the Config Service only observes a new commit when rolling back: no approval or other record of the change is created, besides a new Commit entity.

### Advantages

* The whole `/clusters/<cluster-name>` directory, i.e. the full cluster configuration, can be rolled back.
* The rollback process can be modified to roll back more limited scopes too if needed.
* Git versioning is used directly for tracking configuration changes.
* The git diff can be used to easily show the full extent of the changes and can be used to preview the change.
