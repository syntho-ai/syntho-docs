# Contributing guide

## Introduction

Following the guidelines below will help you and the team members of this project to have a structured and uniform approach towards your contributions on this project, addressing issues, assessing changes, and helping you finalize your pull requests.

## Getting started

### Quick start

* Make sure you have a Azure Devops account and access to the Syntho organization.
* Make sure you have a GitHub account for our open source projects.
* Clone the syntho-engine repository from Azure Repos.
* Install pre-commit for commit validation
* See README.md for further development steps

Please see the [Azure DevOps sprint board](https://dev.azure.com/syntho/Syntho%20Engine%20Core/_sprints/directory) and backlog for functionality that is yet to be or being developed.

We use a form of Trunk-based development for all our project with the following branches:

* master
* feature
  * feature/<username>/<feature_name>
* release
  * release/<sprint_name>

Features should be developed in separate features branches. Hotfixes can be committed directly to the master branch (or trunk), since every commit or PR into the master branch will trigger the build validation. If a commit fails the build validation, your first task is to debug and resolve this issue first so that the build validation succeeds again.

### Commits best practices

Best practices for commits are:

* Contain related changes
* Commit often
* Don't commit half-done work
* Test code before commit
* Write good commit message (Imperative style, with additional bullet points if necessary)

### Releasing

Each repo will have its own release branch under ```/release/***``` in Git. To trigger the release pipeline, a new release branch can be created in Git and then pushed to the Azure DevOps repo. To do this in the git command line, run the following:

```[sh]
git pull origin master # checkout lastest master commit
git checkout -b release/{release_number}
git push origin release/{release_number}
```

Check existing release branches to see what the next release number should be (most cases incremental from 001). **Before running this, check whether the version has been correctly upgraded**. In some cases, the release pipeline will trigger an error if a release version already exists (this is the case for releasing Python packages). Make sure you upgrade the version number to the next version before triggering a release.

### Azure DevOps commit linking

To link commits directly to Azure DevOps items (work item, bugfix) include #(number of work item or bug) in your commit.

Example

```sh
git commit -m "Implemented specific work item for item #4"
```

Read more about this in the following [link](https://docs.microsoft.com/en-us/azure/devops/notifications/add-links-to-work-items?view=azure-devops).

### Testing

Every new feature should be properly documented and well tested. For reproducibility, unit and/or integration tests should be included in new features.

For this package, tests can be run using pytest.

The following command will pick up all tests defined:

```sh
pytest
```

## How to report a bug

### Explain security disclosures first

If you find a security vulnerability, do NOT open an issue. Email info@syntho.ai instead.

### Bug report filing

You can use the following bug report template to report bugs. Bugs can be directly added to the sprint board on relevant user stories.

> When filing an issue, make sure to answer these three questions:
>
> 1. What did you do?
> 2. What did you expect to see?
> 3. What did you see instead?

## Code review process

### Who reviews it?

Each feature will be merged into the master branch will need to have at least two reviewers. Because of the current team size, one reviewer next to yourself is also accepted.

### Who needs to sign off before it’s accepted?

At least 1 other developer needs to approve the feature in order to accept the pull request. In some cases, this number can be higher depending on the importance of a feature.

### How can contributors get commit access?

Commit access is related to project access in Azure DevOps. Once you have access to a certain project, you will be able to commit to certain branches.

### How and when does the reviewer look at pull requests?

If the reviewer is tagged properly in the pull request, the reviewer will be notified of this via email. The reviewer will look at the following things:

* Code readability.
* Implemented tests.
* Whether the branch has followed the requirements properly.

### How is feedback given?

Reviewers and creators of pull requests can exchange messages in the pull request itself in Azure DevOps. The reviewer should add his or her feedback in the pull requests comments and let the creator of the pull request know whether they accept the feature or not after the given feedback, or if the feature still needs some work. Each assigned reviewer either approves, approves with suggestions, wait for the author or reject a feature. In some cases, a reviewer can decline to review at all.

## Code, commit message, and labeling conventions

To streamline the release process, we implemented commit message conventions to be able to automatically determine the release version (using semantic-release for example). Read more about this topic [here](https://github.com/semantic-release/semantic-release).

### Explain if you use any commit message conventions

We adhere to the [Angular Commit Message Conventions](https://github.com/angular/angular/blob/master/CONTRIBUTING.md#-commit-message-format).

### Explain if you use any labeling conventions for issues

No specific labeling for issues has been implemented yet.
