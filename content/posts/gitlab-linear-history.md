+++
title = 'Enabling Linear History Workaround: Minimal Integration with Gitlab Merge Trains'
date = 2024-04-08T17:07:30Z
draft = false
author = 'Sébastien'
tags = ['Gitlab', 'DevOps', 'CI', 'Git', 'Monorepo']
categories = ['DevOps']
+++

## Introduction

If you have already fully adopted linear history you may have encountered challenges when merging code on Gitlab, especially in highly active repositories like mono repos. One particular challenge arises from Gitlab's behavior, as it requires manual rebase and re-running the CI pipeline when merging if there have been any changes on main. In this blog post, I will provide a quick workaround to overcome this issue on Gitlab. Our focus will be on achieving linear history with minimal integration using Gitlab Merge Trains, without delving further into the concept or its full capabilities. Let's explore this workaround and ensure you can maintain a linear history on Gitlab without the pains of manual rebasing during merges.

## Understanding the Challenge
When linear history is enabled on Gitlab, it enforces that a branch can only be merged if there have been no previous merges. If there have been previous merges, Gitlab forces manual rebasing and re-running the CI pipeline until a merge can be performed. This can make merging on a busy repository a harrowing experience.

## Desired behavior
In order to achieve our desired behavior and mimic the experience of working with linear history on other platforms, we have the following requirements:

- A successful Merge Request Pipeline must have run.
- The Merge Request commits can be rebased and fast-forwarded onto the main branch without conflicts.

When these conditions are met, we should be able to merge the changes into the main branch without the need to manually rebase or re-run the Merge Request pipeline.


## The Workaround
To enable linear history with minimal integration on Gitlab and achieve the desired behavior, follow these steps:

1. Enable Gitlab Merge Trains for your repository.
2. Set the merge strategy to fast-forward.
3. Implement a simple Gitlab CI pipeline that satisfies the requirements for the Merge Train pipeline to proceed with merging.

Here's an example .gitlab-ci.yml configuration you can add to your repository's root directory as a starting point:

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
    # this is just a dummy pipeline fulfilling the requirement of a successful Merge Train Pipeline
    - echo "Successful Pipeline..."
```
By following these steps, you can ensure that Merge Request commits can be seamlessly rebased and fast-forwarded onto the main branch if there are no conflicts, even in highly active repositories. All you need to do is add your Merge Request to the Merge Train and wait for the dummy pipeline to run.

Please note that Gitlab Merge Trains require a Gitlab Runner and the ability to run a Gitlab CI pipeline. External pipelines are not supported.

## Conclusion
With this minimal integration workaround, you can enable linear history on Gitlab while side stepping the need for manual rebase on Merge Requests.

Remember, Gitlab's Merge Trains offer comprehensive capabilities that you can further explore in the future, should you decide to fully adopt them. I believe that Merge Trains provide a valuable opportunity to shift your pipeline left, potentially eliminating the occurrence of a failed main builds entirely.

Enjoy the benefits of a linear history on Gitlab. Happy coding!