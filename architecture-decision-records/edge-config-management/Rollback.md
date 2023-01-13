# Rollback cluster state

## Scenario

Configuration mistakes and buggy new versions are bound to happen.  Knowing that it is possible to roll back to a known good version or configuration provides immense peace of mind to developers, site operations and end users.

## Requirements

1. The rollback process should work on a cluster level, and should be able to bring back the cluster configuration to a known good state. The cluster configuration encompasses the whole state of the cluster, but here is scoped to only the state of the cluster as represented in the GitOps repository. Determination of the state to which the cluster should be rolled-back will be done by the operator (i.e. it is not automatically determined through automation).

2. Rollback should complete reasonably fast, ideally no slower than a regular configuration change.

3. The rollback process should not have an impact on cluster management after the rollback has been completed, i.e. the cluster should be operatable as per usual after a rollback, without need for special considerations. In effect, rollback should just be another configuration change, instead of a special disaster recovery mode or similar.

## Background

At least two generic approaches exist for rollback:

1. Using versioning to create immutable artifacts which can be rolled back to
2. Having a mechanism that can restore the previous configuration state from some kind of persistent storage, like the git history

As the configuration of the cluster is mainly stored in the GitOps repository, the scope of the rollback must encompass the GitOps repository. The configurations outside of the GitOps repository are outside of scope of this document, but should also be taken into consideration.

## Rolling back and handling global directories

The GitOps repository consists of the `/clusters` directory, that houses cluster-specific configuration, and other directories like the `/apps` and `/infra` directories that contain configurations that apply to multiple clusters (global configuration).

**In order to fulfill requirement 3, it is not possible to modify the global configuration when rolling back a single cluster, thus approach 1 should be used for those directories.** This means that the `/apps`, `/infra` and other global directories should be append-only, i.e. existing application versions are immutable and only the creation of new versions is permitted, and all other configurations, like configmaps, must be versioned as well.

The clusters configuration in the cluster directory however can be rolled back as explained below.




## Solution

Rollback can be done by changing the contents of the `clusters/<cluster-name>` directory to a known good state. i.e. via git history

### Practical implementation

1. Determine the known good version of the `clusters/<cluster-name>` directory by determining which git commit was at HEAD of the GitOps repository at the known good state. A more thorough approach would be to check which commit was reported as deployed onto the cluster at the known good state with observability tooling.
2. Fetch & check out the GitOps repository to a working directory. Run
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

### Benefits

* The whole `/clusters/<cluster-name>` directory, i.e. the full cluster configuration, can be rolled back.
* The rollback process can be modified to roll back more limited scopes too if needed.
* Git versioning is used directly for tracking configuration changes.
* The git diff can be used to easily show the full extent of the changes and can be used to preview the change.
