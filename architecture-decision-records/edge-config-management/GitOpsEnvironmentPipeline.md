# Promoting changes to Environments using GitOps Repositories

## Problem
How do we want to promote changes in our GitOps repository from dev to quality to prod to ensure code cannot skip one of the gates?

## Foundation, Requirements and Constraints 
- Based on the ADR [GitOps: Representation of Environments](./GitOpsEnvironments.md)
- We want to ensure that we can promote changes between different repositories linked to each one different environment
- We want to ensure that only changes to selected folders are promoted
- We want to support a workflow where developer can work in a feature branch, test changes from this feature branch on dev, close the development with a PR with triggers build pipeline that can deploy this changes to the dev stage and afterwards to qual and prod.
- We want to preserve the relevant history on the environments GitOps Repo.
  - Relevant includes:
    - Changes from the GitOps Config Service. (Those are limited to the cluster folder and will not change anything in apps and infra)
    - Release Notes from the pipelines for the folders apps and infra
  - Non-requirements
    - The full development history is not needed in the prod gitops repository, as long as the full development history can be traced from commit hash links.
- Assumption: We want to work with branch filters to ensure that feature and main branches can be deployed to dev and hotfix and main branches can be deployed to all environments. 

# Alternatives:

## Environment Repositories with feature branches
There are 3 different repositories: Dev, Qual, and Prod.
We do all development on dev in feature branches, after the PR and merge to main we deploy these changes with a pipeline(s) to the different stages.

This could, for example, look like this:

![EnvironmentRepos](./images/GitOpsEnvironmentPipeline/EnvironmentRepos.drawio.png)

- We would only need 2 pipelines or pipeline step to promote from dev to qual and from qual to prod. 
  - And it is easy to ensure that changes are only moved without skipping an environment
  - Or we use a pipeline/release approach as describe in "Source Repositories and Environment/Deployment Repositories"

- We would need to introduce a couple of additional policies for the dev git ops repository that are not needed on qual and prod. 

### Challenges:
- How to test changes from the feature branch? 
  - As the edge devices and the configuration service are configured to listen to main branch on the dev repo changes to a feature branch will not be applied. 
  We could mitigate this with a separate dynamic environment spun up for the feature branch or with reconfiguring Config server and flux to listen to our feature branch.

- How could we ensure that only changes from Infra and Apps folders are promoted?
  - Our pipeline could simply ensure that only these folders are copied and pushed to the next environment.
- How can we ensure a clean git history if this server is used for configuration changes and development changes? 

## Source Repositories and Environment/Deployment Repositories
We introduce a fourth repository which represents the source git ops configuration.
This approach is a bit like all other repositories where we have a single repository for our code and deploy this to the different environments (And we simple see the git ops repo as part of the environment).
Additionally, it distributes the responsibilities of the dev git ops repo to two independent repositories: one that are used to test and apply changes in dev and one repository that is used to support the development process with branch policies, linked work items and a clean git history.

This could, for example, look like this:

![EnvironmentRepos](./images/GitOpsEnvironmentPipeline/SourceRepo.drawio.png)

- In this setup tests on dev are easy to start as each push to a feature branch gets deployed to our dev stage. (But parallel development on dev could still create demand for different environments)
- All environments are configured and setup the same way 

# How could a deployment look like (works for both alternatives):
  - For example, a pipeline could select the folders apps and infra and create a deployment zip (includes all sub folders and files, without git files)
  - A release could organize the deployment of this zip to the different environments
    - In this way we get the good overview of releases and can in addition customize the git commit to ensure we have a easy to read message.
    - The combination of releases and pipeline makes it easy to link changes to a user story 
    - We can ensure that all stages get the same changes, and only changes to Apps and Infra Folder are promoted

- How will the pipeline work? 
  - [ ] Fork / Upstream branches or copy/paste?
  - [ ] Commits to main or PR for each environment?
  - [ ] How much of the git history is important to be available on prod?
  - [ ] Should it be possible to remove files from the Apps and Infra folders?

# Open points / Side questions
- [ ] How can we ensure that we only promote configuration that is using images that are already promoted to the target environment? 
  - We could break our environment if we promote a configuration without the corresponding image available!

# Conclusion:

| Feature | Environment Repositories |Source Repositories |
|--------------|--------------|--------------|
| Clean Git History | No, Git will contain PR for features and Cluster Changes |        Yes, Git Source will only contain Feature Changes |
| Easy to Link Changes to Tasks/User Stories | Yes, can be done via release and pipeline | Yes, can be done via release and pipeline |
| Easy to Test changes in Dev Environment | No, needs adjustments for testing | Yes, works without adjustments if parallel tests aren't an issue |
| clean pipeline that allows to see deployments to different environments | Yes, Combination of pipeline+releases in Azure DevOps | Yes, Combination of pipeline+releases in Azure DevOps |
| Could independent development teams work on different apps ? | yes, but they would share a repo and pipelines, additional coordination between the teams would be necessary | yes, each team could have its own source repo and pipelines. Coordination would only be necessary for `/infra` which could an own repo and team itself  | 
| other points / considerations | ... | ... |
