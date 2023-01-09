# Connection to our Git Repo

## Problem
We need a way to ensure that our Config Service and Flux on our Edge device have access to out git repository

## Context
As we are going to use GitOps we need access to our GitOps repository. Azure DevOps Pipelines, the Config Service API and the edge modules need to authenticate against the repo.

## Decision Drivers
We want to be able to limit the available scopes for our clients.
We want to be able to rotate the access token, secrets, pat that are used to access git.
We want to control the access for Azure DevOps, Config Service, and the edge modules (All edge modules as one not each module separately) individual 

# Options:
## Personal Users 
One of the Developers uses his account to provide access to the git repositories. 

Pro:
- Easiest and quickest solution

Con:
- Access is coupled to the developer account and would need to be changes for example if the developer leaves the project.
- PAT token also has access to all repos the developer has access to
- PAT token can be used to impersonate said developer by other developers.

## Azure Service Account
We register an Azure Service Account that is used to manage the access to the git repositories.

Pro:
- Dedicated DevOps User that can be managed by a set of developer to create or remove access to our git repos.
- Independent from single developer accounts
- We can manage multiple PATs with on Service Account
- Service account can be given access only to the GitOps repo, limiting the access of the PAT token to that specific repo.

Con:
- Needs additional time to request and setup the Service Account

Do we want to use ssh keys or PAT Token?
- With PAT tokens a specific scope on which the token is allowed to operate on can be specified, for example, a `read only` scope for repositories. `SSH` keys can't be configured in such a way.

## App Registrations
We could investigate if we could create one or more App Registrations, add those as devops users and use those to grant access to git.
It is unclear if this would work and if additional DevOps licenses would be needed.

Pro:
- Quick setup, managed by our AdmAccounts
- easy key rotation

Con:
- unclear if this would work

We should use a different service account for each environment.
Due to the limitations of the PAT token, in which you can't limit the access to specific repositories within an organization.
A PAT token is created per organization, however. But we can limit the access of the service account to specific repositories to give them only the necessary access.
