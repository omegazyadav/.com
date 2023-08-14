---
title: "Github Action Reusable Workflows"
date: 2023-05-25
author: Yadav Lamichhane
description: "Manage Reusable workflow with Github Actions"
tags:
- Linux
---

# GitHub Reusable workflows

In this blog post, we will be exploring the Github Reusable workflows and how we can leverage them. The primary purpose of reusing workflows is to avoid code duplication when we have similar repositories with similar workflows. This makes workflows easier to maintain and allows you to create new workflows more quickly by building on the work of others, just as you do with actions. Workflow reuse also promotes best practices by helping you to use workflows that are well-designed, have already been tested, and have been proven to be effective.

![](https://cdn-images-1.medium.com/max/2462/1*8gQuw2Y5WnSkCN8HnJf7Ww.png)

The above diagram shows service deployment workflow is kept inside ga-flows-central repository while child services like service1 service2 service..n is calling central workflows.

Since we are trying to have cross-repository access with both being private repositories, we need to enable access with the following steps:

**Managing access of central reusable workflows to private repositories**

* On GitHub, navigate to the main page of the repository where your reusable workflow resides

* Under your repository name, click **Settings**.

* In the left sidebar, click **Actions**, then click **General**.

* Under **Access**, choose the access settings:

* **Accessible from repositories in the “cloudhonk” organization**

* Click **Save** to apply the settings.

![](https://cdn-images-1.medium.com/max/2000/1*6-7Jm41ZV-LhYAYMObfNHQ.png)

For experimentation purposes, we have created a reusable workflow in ga-flow-central repository whereas calling workflow is defined in ga-flows-child repository.

![](https://cdn-images-1.medium.com/max/2000/1*OVZObQqbUtGBGDVmUBMiRA.png)

In this post, we will be covering the following topics:

* Reference of Reusable workflow from another repository

* Passing Repository Secrets Variables

* Passing Environment Variables

**Reference of Reusable workflow from another repository**

Here we will use a simple workflow that will print the git version used in the workflow. The content of the reusable workflow in ga-flows-central repository is as follows:

    name: Reusable Github Actions - Demo

    run-name: Github Reusable Workflow

    on:
      workflow_call:

    jobs:
      called-workflow:
        runs-on: ubuntu-latest
        steps:
          - name: Check out Repository
            uses: actions/checkout@v3

          - name: Print the git version
            run: |
              git --version
> **Note: For a workflow to be reusable, the values for on must include workflow_call:**

Now we can call the reusable workflow defined in ga-flow-central repository in ga-flows-child with uses keyword:

    name: Reusable GA Workflow Demo

    on:
      push:

    jobs:
      calling-workflow:
        uses: cloudhonk/ga-flow-central/.github/workflows/reusable.yml@feat/git-version

reusable.yml file on ga-flow-central the repository was configured inside feat/git-version branch so we need to append @feat/git-version on called workflow.

    uses: cloudhonk/ga-flow-central/.github/workflows/reusable.yml@feat/git-version

![Fig: ga-flow-child repo calling ga-flow-central workflows to print git version](https://cdn-images-1.medium.com/max/2000/1*vUoG8jZ0bwetm0Ght5oRyQ.png)*Fig: ga-flow-child repo calling ga-flow-central workflows to print git version*

**Passing Repository Secrets Variables**

We can use the inherit keyword to pass all the calling workflow’s secrets to the called workflow. This includes all secrets the calling workflow has access to, namely organization, repository, and environment secrets. The inherit keyword can be used to pass secrets across repositories within the same organization, or across organizations within the same enterprise.

In this case, we will be creating a personal access token and keeping the token as a secrets variable in ga-flow-childrepository instead of ga-flow-central . This will enable us to override a secrets variable dynamically as per our use cases in different repositories.

Below mentioned workflow should clone the private repository with the use of a PAT token and list all the files and directories present in it.

Called workflow:

       - name: Clone the Github Repository
         run: |
           git clone https://${{ secrets.ORG_GITHUB_TOKEN }}@github.com/cloudhonk/ga-flow-central
           ls ga-flow-central

Calling workflow:

    jobs:
      calling-workflow:
        uses: cloudhonk/ga-flow-central/.github/workflows/reusable.yml@feat/reusable-workflows
        secrets: inherit

![](https://cdn-images-1.medium.com/max/2000/1*GurZd5fw__vho5fEANYEMg.png)

**Passing Environment Variables**

We can override the environment variables defined in the called workflow with user-specified values in the calling workflow. This is useful when you want to override the environment variables based on the repository.

Here, DAY_OF_WEEK defined in gal-flow-central repo has default value as “Tuesday”. What if we wanted to override that value with something else, lets say with “Saturday”. For this case we have to keep DAY_OF_WEEKrepository variables in ga-flow-child repo and reference it with with keyword.

    with:
          DAY_OF_WEEK: ${{ vars.DAY_OF_WEEK }}

**Workflow in ga-flow-central repository**

    name: Reusable Github Actions - Demo

    run-name: Github Reusable Workflow - ${{ github.actor }}

    on:
      workflow_call:
        inputs:
          DAY_OF_WEEK:
            required: true
            type: string
            default: "Tuesday"

    jobs:
      called-workflow:
        runs-on: ubuntu-latest
        steps:
          - name: Check out Repository
            uses: actions/checkout@v3

          - name: Print the environment variables
            run: |
              echo ${{ inputs.DAY_OF_WEEK }}

**Workflow in calling repository**

    name: Reusable Github Actions Workflow Demo

    on:
      push:

    jobs:
      calling-workflow:
        uses: cloudhonk/ga-flow-central/.github/workflows/reusable.yml@feat/reusable-workflow
        with:
          DAY_OF_WEEK: ${{ vars.DAY_OF_WEEK }} # Create a Repository Variable for DAY_OF_WEEK
        secrets: inherit

![](https://cdn-images-1.medium.com/max/2000/1*10uC_P0buy68yvoKafwmpA.png)

Here, the value for DAY_OF_WEEK has been overridden from “Tuesday” to “Saturday”.

This is all about GitHub Actions Reusable Workflow, we have been using it for almost all the repositories for quite some time now. If you have any questions related to this, please feel free to drop comments on the post.

Thank you.

**References**

[Reusing workflows - GitHub Docs](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

[GitHub - cloudhonk/ga-flow-central](https://github.com/cloudhonk/ga-flow-central)

[*GitHub - cloudhonk/ga-flow-child*](https://github.com/cloudhonk/ga-flow-child)

