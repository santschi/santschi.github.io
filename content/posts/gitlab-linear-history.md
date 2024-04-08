+++
title = 'Draft - Linear History on a Busy Repo on Gitlab'
date = 2024-04-08T17:07:30Z
draft = false
author = 'Sébastien'
tags = ['gitlab', 'git', 'CI/CD']
+++
# Gitlab and Linear History
Gitlab supports linear history out of the box. But there are severe limitations. On a regular Gitlab repository
you have to rebase a Merge Request every time that main changes. This is done by clicking the handy UI button if it can be done without conflicts. But
this re-triggers the entire Merge Request pipeline.

On a small or infrequently used repository this is not a problem since there is little contention regarding merging. But on bigger repos,
especially monorepos, this can make merging nearly impossible. `A` merges a change, `B` - `Z` all have to rerun their pipeline. The person with the shortest pipeline
(or the fastest finger) merges next. Everyone else back to the starting position. This is not a tenable state for an active repository.

## Desired behavior
If you are coming from a different platform and have been using linear history it is likely that your infrastructure is designed to work with a classic Pull Request.
To make the transition easier we try to approximate the behavior on gitlab:

Our desired behavior is that
- if there was a **successful** Merge Request **Pipeline** and,
- The Merge Request commits can be **rebased** and fast-forwarded onto main **without conflict**

then we should be able to merge to main, without having to rebase manually or re-run the MR pipeline.

## Merge Trains
Gitlab is strongly pushing their concept of a Merge Train. In simple terms this is a queue of Merge Requests which have indicated that they are ready to be merged to main.
Each MR (or at least the corresponding Merge Train Pipeline) is aware of all the MR preceding it in the queue.
While I do not want to deep dive the Merge Train concept here, the underlying principle is that each MR runs a Merge Train Pipeline where all preceding MR's in the queue are already
applied to main. If all proceeding Merge Train pipelines succeed and the current MR's Merge Train pipeline succeeds as well Gitlab rebases and fast-forward merges the MR onto main.
There is quite a bit of nuance here, and there is also the different merge strategies to be aware of. But for our purposes that's all we need to know.

### External Pipeline do not Work for Merge Trains
One issue when migrating to Gitlab is, that Merge Trains do not seem to support pipelines triggered by webhook. While the MR and main pipeline can be triggered by webhook without issue,
the MT pipeline has to be on Gitlab.

## Using Merge Trains to Rebase and FastForward Only
Since Merge Trains are the only mechanism Gitlab offers that does the Rebase for you, we use them to get the desired behavior of being able to merge non-conflicting MR's.

**Enable Merge Train** for the repository in question. Make sure the chosen **merge strategy is fast-forward**. There needs to be a Gitlab CI pipeline that succeeds for the Merge Train to proceed on merging so we add the following `.gitlab-cy.yml` to the root directory:

```yaml{hl_lines=[9]}

stages:
  - dummy

satisfy-gitlab:
  stage: dummy
  image: busybox:3.15
  rules:
    - if: $CI_MERGE_REQUEST_EVENT_TYPE == "merge_train"
      when: always
  script:
    - echo "Successful Pipeline..."
```


## Last Words
We recreated a classic linear history repository on Gitlab. But we also discarded a lot of powerful tools Gitlab has to offer along the way.
While I do think this to be a good intermediary solution when migrating to Gitlab with existing Infrastructure it comes at the cost
of not using Merge Trains as the powerful tool they are. If Merge Trains are used appropriately they can help immensely in ensuring
the continuous quality of the main branch and in turn this can enable you to truly deploy (more) continuously with a lot more confidence.
The ability to run a pipeline on the identical state of a repository as it will be when merged, without having to touch the actual main
branch enables testing with very high confidence in a highly concurrent environment. And the best part is, that it can be fully done in parallel,
with very little overhead if the quality of the MR pipeline is high enough.