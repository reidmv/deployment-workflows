# Gitflow, Agile, and Puppet

## HOW TO USE THIS DOCUMENT

*This is NOT a proposed workflow, nor a proposed solution or solution(s). It should be used as an aid in thinking through workflow problems in Puppet and r10k.*

## Problem Description

Suppose RG Bank (a hypothetical entity) uses Puppet to manage their Business Operations application infrastructure. This infrastructure consists of baseline configuration and 11 individual applications, some single-tier apps, some multi-tier apps. These applications are developed on RG Bank owned and managed infrastructure called Lab. Two separate Managed Service Providers (MSPs) then consume RG Bank's Puppet code to deploy and manage the apps.

RG Bank has XX deployment tiers, or environments, which changes are pushed to in order as part of the deployment process. These deployment tiers are:

1. development
2. test
3. prestage
4. stage
5. production

Each of the 11 applications RG Bank manages have their own Software Development Lifecycle (SDLC) cadence. Changes to each app are first deployed to development. Those changes are then promoted to test, to prestage, and so on until reaching production.

Different applications are developed and deployed at widely varying cadences. Baseline configuration changes to services like ntp and dns move very quickly through development all the way to production. Changes to one of the apps, FTO, move more slowly though. Changes to this app might be left in the "test" deployment tier for a long time to bake-in, and make sure there are no problems after long periods of operation.

Frequently, there is a need for changes to baseline configuration or rapidly-developing apps to come up from behind an existing change already baking in the test deployment tier, and pass through on to prestage or production.

Puppet manages the configuration on all the servers in every deployment tier. The traditional r10k workflow accounts for this, but does not account for multiple parallel apps represented by the codebase being deployed to SDLC deployment tiers at differing rates.

This project aims to suggest possible workflows separating development of Puppet code from deployment of Puppet code. Tools used include using Git, Hiera, and R10K.

## Feature branch merge deployment strategy

This approach uses Git to keep features in branches, independently referencable until they've been merged into every deployment environment.

The downside to this approach is it is heavier on the use of Git, and when things go wrong they have to be fixed using Git.

### Merging strategy

* Ordered, named branches are maintained in Git where each branch corresponds to a SDLC stage
    * There are _n_ total SDLC branches in the ordered SDLC promotion path
    * "integration" is the 1st branch
    * "production" is the _n_-th branch
* Features are developed in feature branches forked from production
* Feature branches are iteratively merged into the ordered SDLC branches, starting with integration
    1. The feature branch is merged into integration: `git checkout integration; git merge feature`
    2. The feature branch is merged into the integration+1 SDLC branch: `git checkout [integration+1]; git merge feature`
    3. The feature branch is merged in order into each of the remaining non-production SDLC branches
    4. The feature branch is merged into production
* If changes are made to a feature branch before it is merged into production, its promotion target resets back to integration
* Feature branches are deleted after being merged into production
* After a feature branch has been merged into integration it should not deleted until merged to production. If the feature is abandoned, the feature branch commits must all be reverted (in the feature branch), and the "abandoned" feature branch then promoted through to production.

### Actions

#### Promote a feature branch

Promoting a feature branch means merge it to the next SDLC branch. The "next" SDLC branch is the first SDLC branch, starting from integration, which does not contain the feature branch's HEAD commit.

#### Make a change to a feature branch

"Fixing up" a feature branch is making any new commit to it. An implication worth noting is that making a change to a feature branch which has already been promoted "resets" that feature branch's "next" SDLC branch back to integration.

#### Resolve a merge conflict

Unfortunately, resolving merge conflicts may still be necessary on occasion. When this happens it will likely require git work and human judgement to resolve. It is difficult to suggest an algorithmic approach to resolution.

## Profile versions deployment strategy

A common strategy is to build parameterized classes and use Hiera data to activate different configurations based on a node's SDLC tier. In this model, Puppet code contains simultaneously multiple different configurations, and which is deployed is selected by Hiera.

The downside to this approach is that the code becomes more complicated.

### Puppet Code

```puppet
class profile::fto (
  Integer[1, 2] $profile_version = 2,
) {

  case $profile_version {
    1: { include profile::fto::v1 }
    2: { include profile::fto::v2 }
    default: { fail("cannot configure ${title} profile_version ${profile_version}."
  }

}
```

### Hiera

Hiera contains per-SDLC environment settings indicating which profile versions should be used for each application. E.g.

*deployment\_tier/production.yaml:*

```yaml
---
profile::fto::profile_version: 1
```

*deployment\_tier/development.yaml:*

```yaml
---
module::profile_fto::version: "1.1.0"
module::profile_fh::version: "4.0.0"
module::profile_bigfix::version: "2.0.0"
```

### Create a new profile version

1. Create a feature branch for the new profile change
2. Modify the profile
    * Edit the `profile_version` parameter validation to include the next version number
    * Edit the `profile_version` parameter's default value to be the newest valid version number
    * Edit the case statement, adding the new version number and `include` block
    * Edit the `profile_version` parameter validation and case statement, removing the oldest version(s) that are no longer deployed in any SDLC environment
3. Modify hiera
    * Edit `common.yaml` and set the relevant `*::profile_version` parameter to the _currently deployed_ version number. If multiple versions are currently deployed set it to the lowest currently deployed number
    * Edit each `deployment_tier/*.yaml` file and make sure the relevant `*::profile_version` parameter is set to the correct value for that SDLC environment
4. Merge the feature branch

### Promotion workflow

1. Create a feature branch for the new version deployment
2. Edit the `deployment_tier/*.yaml` for the SDLC environment you want to promote a version to and set the relevant `*::profile_version` parameter
3. Merge the feature branch
