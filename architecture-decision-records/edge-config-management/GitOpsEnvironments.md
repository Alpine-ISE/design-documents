# GitOps Repositories and Environments

## Problem
How do we want to represent the different state of each environment and promote changes between different environments?

## Context
- We want to use independent and isolated environments to separate development, test and production code:
- Each environment has its own Azure Subscription and resources.
- There are Edge devices that use K3S with Flux to pull the cluster configuration from a central git repository.
  - The repository structure is described in the [sample gitOps repo] (https://github.com/buzzfrog/contoso3)
  - The GitOps repository contains 3 base folders:
    - `apps`: Contains the base setup of each application. Each application can contain one or more deployments, services, configMaps etc.  Only Developers/Ops will update content within this folder.
    - `infra`: Contains the base setup of the infrastructure the applications need to run on the cluster eg. MQTT Broker.  Again, only Developers/Ops will update this.
    - `clusters`: Contains a directory for each cluster which is connected to this GitOps repository.  Initial folders must be created by the Ops team as a new cluster is onboarded (i.e. when a new Edge device is provisioned).
  - Therefore, changes from `/apps` and `/infra` should be promoted from Dev to Qual to Production, whereas changes in `/clusters` should not be promoted as they are environment dependent.
    - [ ] We want to ensure that `/apps` and `/infra` should always be promoted as a bundle including all changes. To stage changes we could use feature branches and PRs.
    - [ ] We want to be able to promote selected apps or infra projects individual, this would mean that we want to be able to promote selected sub folders from `/apps` and `/infra` to allow small and flexible updates.
    - We decided that we want to ensure that our workflow for software and gitops are similar and we are able to use the same mechanics to work, review and promote results. 
- There are several ways we can represent & differentiate the environments in code (e.g. different branches, folders, or repositories) which we shall explore in this ADR.

## Decision Drivers

1. Ensure that all code goes through test (Qual) before being released to Production.
2. Ease with which problems observed in Production can be traced back to the original code or config changes.
3. Simple to understand - reducing human error, easing developer/ops onboarding & productivity.
4. No loss of change history.
5. Does not degrade when scaled to hundreds of devices.

# Options:
## Represent in different branches
The idea is to have different branches for each environment (main, dev, qual) and configure flux on the device to listen to the specific branch.

Pro:
- easy to setup
- easy to compare between branches
- branches can use branch protection to limit changes to specific sources

Con:
- many branches (for features, for stages, for prs, ...)
- no real separation between environments (changes to the repo cant be tested)
- Addition effort to ensure a client cant only connect to a given branch (to ensure that a client can only load information from its environment)

## Represent in different folders
The idea is to have different folder for each environment (/prod, /qual, /dev) and configure flux on the device to listen to the specific folder.

Pro:
- easy to setup
- easy to promote (copy&paste) and merge with a PR

Con:
- difficult commit history (as you have any change from any environment)
- no real separation between environments (changes to the repo cant be tested)
- addition effort to ensure a client cant only connect to a given folder (to ensure that a client can only load information from its environment)
- multiplication of folders (less scalable)
- copy&paste: humans make mistakes - not all are detected in a PR ;-)

## Represent in different repositories
The idea is to have different repositories for each environment (gitops_prod, gitops_qual, gitops_dev) and configure flux on the device to listen to the specific repository.

Pro:
- real separation between environments
  - changes to GitOps repo can be in dev
  - each environment knows only its state
- clean history, clean folder structure

Con:
- additional effort to promote changes from one repo to another environment repo
