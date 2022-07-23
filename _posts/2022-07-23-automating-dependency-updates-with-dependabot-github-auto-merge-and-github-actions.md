---
layout: post
title:  "Automating dependency updates with Dependabot, GitHub auto-merge, and GitHub Actions"
share-description: This tutorial shows how to set up a workflow to automatically merge low-risk dependency updates while streamlining the process to fix and manual test higher-risk dependency updates.
tags: dependabot automation dependencies github-actions tutorial
---

I once worked on a team that had a quarterly reminder on Slack to update all dependencies across every project. Our main goal was to prevent our dependencies from getting too old and becoming a nightmare to update later. We also wanted to rip the benefits of new features, bug fixes, security, and performance patches in the frameworks and libraries we used as soon as possible.

Most of the time, our dependencies would have a patch or minor update which didn't break our codebase. However, on a few occasions, we had a major version update that required changes due to backward incompatibility.

To assist us in keeping our dependencies in check, we enabled Dependabot in our projects. [Dependabot](https://github.com/dependabot) is an automated dependency update tool at the disposal of any GitHub repository. When enabled, it scans the project in search of outdated dependencies and creates pull requests to update them.

Dependabot was a great addition to our tooling to streamline the process of updating dependencies. But then, with Dependabot creating pull requests every day, we quickly became overloaded with the number of changes we had to merge to our main branch. Since the vast majority of dependency updates were patch or minor versions, representing a low risk to the project, I always wondered if we could automate the process of merging these changes.

Eventually, I left the company, but I remained curious about how we could set up a workflow to automatically merge the low-risk dependency updates.

While maturing this idea, I came up with the following flowchart.

![Dependency update flowchart](/assets/img/posts/2022-07-23-automating-dependency-updates-with-dependabot-github-auto-merge-and-github-actions/dependency-update-flowchart.png)
<!-- https://dreampuf.github.io/GraphvizOnline/#digraph%20G%20%7B%0A%20%20%20%20dependabot_creates_new_pr%5Blabel%3D%22PR%20created%20by%20dependabot%22%20shape%3D%22rect%22%5D%0A%20%20%20%20is_build_passing%5Blabel%3D%22Is%20build%5Cnpassing%3F%22%20shape%3D%22diamond%22%5D%0A%20%20%20%20is_major_update%5Blabel%3D%22Is%20major%5Cnupdate%3F%22%20shape%3D%22diamond%22%5D%0A%20%20%20%20merge_pr%5Blabel%3D%22Merge%20PR%22%5D%0A%20%20%20%20is_production_dependency%5Blabel%3D%22Is%20production%5Cndependency%3F%22%20shape%3D%22diamond%22%5D%0A%20%20%20%20pr_is_waiting_for_fix%5Blabel%3D%22PR%20waits%20to%5Cnreceive%20a%20fix%22%20shape%3D%22rect%22%5D%0A%20%20%20%20waiting_for_qa%5Blabel%3D%22PR%20waits%20to%5Cnbe%20manually%20tested%22%20shape%3D%22rect%22%5D%0A%20%20%20%20fix_build%5Blabel%3D%22Fix%20the%20build%22%20fillcolor%3D%22%23ffcccc%22%20style%3D%22filled%22%5D%0A%20%20%20%20test_pr%5Blabel%3D%22Perform%20manual%20test%5Cn%26%20fix%20if%20any%20issues%20arise%22%20fillcolor%3D%22%23ffcccc%22%20style%3D%22filled%22%5D%0A%20%20%20%20trigger_ci%5Blabel%3D%22Trigger%20the%5CnCI%20pipeline%22%5D%0A%20%20%20%20%0A%09dependabot_creates_new_pr%20-%3E%20trigger_ci%0A%09trigger_ci%20-%3E%20is_build_passing%0A%09is_build_passing%20-%3E%20is_major_update%5Blabel%3D%22Yes%22%5D%0A%20%20%20%20is_build_passing%20-%3E%20pr_is_waiting_for_fix%5Blabel%3D%22No%22%5D%0A%20%20%20%20is_major_update%20-%3E%20merge_pr%5Blabel%3D%22No%22%5D%0A%20%20%20%20is_major_update%20-%3E%20is_production_dependency%5Blabel%3D%22Yes%22%5D%0A%20%20%20%20is_production_dependency%20-%3E%20merge_pr%5Blabel%3D%22No%22%5D%0A%20%20%20%20is_production_dependency%20-%3E%20waiting_for_qa%5Blabel%3D%22Yes%22%5D%0A%20%20%20%20pr_is_waiting_for_fix%20-%3E%20fix_build%0A%20%20%20%20fix_build%20-%3E%20is_major_update%0A%20%20%20%20waiting_for_qa%20-%3E%20test_pr%09%0A%20%20%20%20test_pr%20-%3E%20merge_pr%0A%7D -->

The flowchart details all the steps a pull request (PR) goes through, from when Dependabot opens it to when it gets merged.

The two steps in red are the only manual interventions. All the other steps are automated.

In the vast majority of pull requests opened by Dependabot, patch, or minor updates that don't break the continuous integration (CI) pipeline, the process would be completely automated, and it would look like the following:
1. PR created by Dependabot
1. Trigger the CI pipeline
1. Is build passing? (Yes)
1. Is major update? (No)
1. Merge PR

The rest of this tutorial describes how to bring this workflow to life using Dependabot, GitHub's [auto-merge feature](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/automatically-merging-a-pull-request), and [GitHub Actions](https://github.com/features/actions).

## Step 1: Enabling Dependabot in the repository

We enable Dependabot to a repository by creating a folder called `.github` in the root folder and adding a file named `dependabot.yml` inside.

The most straightforward configuration for the `dependabot.yml` file is described below:

```yml
version: 2
updates:
  - package-ecosystem: "bundler"
    directory: "/"
    schedule:
      interval: "daily"
```

In the configuration above, we are telling Dependabot to use `bundler`, Ruby's package manager, to scan the repository at the root directory daily.

We can learn more about other options to configure Dependabot and the supported package managers by referring to [the official documentation on GitHub](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates).

As soon as we push the configuration file mentioned above to GitHub, Dependabot will perform the first scan of our project and create the first pull requests to update dependencies, unless all the dependencies are already at their latest version.

We can simulate Dependabot opening a pull request by downgrading a dependency and pushing the code to the main branch. After that, we have to trigger Dependabot to scan our project by going to Insights, Dependency graph, Dependabot, and clicking on the link that indicates when was the last check, and finally click on "Check for updates". After that, we should see Dependabot creating a new pull request to update the dependency we just downgraded.

![Trigger dependabot check](/assets/img/posts/2022-07-23-automating-dependency-updates-with-dependabot-github-auto-merge-and-github-actions/trigger-dependabot-check.gif)

## Step 2: Enabling auto-merge in the repository

GitHub allows us to set a list of conditions that, when met, automatically merge a pull request to the main branch.

To enable this feature, we have to go to the repository's homepage on GitHub, click on Settings, and check the checkbox "Allow auto-merge".

To set up the rules in which GitHub will auto-merge pull requests to the main branch, we go to Settings, Branches, and then add a new branch protection rule.

There are many different options to set up these rules, but in this step, we want to set the branch pattern name to "main",  check the box "Require a pull request before merging", check the box "Require approvals", select 1 required approval and click save.

The form should look somewhat like the image below:

![New branch protection rule](/assets/img/posts/2022-07-23-automating-dependency-updates-with-dependabot-github-auto-merge-and-github-actions/new-branch-protection-rule.png)

If we go to one of the pull requests created by Dependabot in the previous step, we should see a button at the bottom of the page saying "Enable auto-merge".

![Enable auto-merge for PR](/assets/img/posts/2022-07-23-automating-dependency-updates-with-dependabot-github-auto-merge-and-github-actions/enable-auto-merge-pr.png)

Let's click on this button and confirm the action in the next step. We should now see a message saying "Auto-merge enabled".

We can now approve the pull request and see GitHub merging it automatically, fulfilling the branch protection rule we created in the previous step.

## Step 3: Creating a GitHub action to review Dependabot pull requests

Once we have the auto-merge feature enabled for our repository, we can create the GitHub action that will approve pull requests opened by Dependabot.

This process will free us from having to enable the auto-merge for the pull request and approve the dependency upgrades we want to automatically merge.

For this, we will create a file at `.github/workflows/dependabot-reviewer.yml` and add the following content to it:

```yaml
name: Dependabot reviewer

on: pull_request_target

permissions:
  pull-requests: write
  contents: write

jobs:
  review-dependabot-pr:
    runs-on: ubuntu-latest
    if: ${{ '{{' }} github.event.pull_request.user.login == 'dependabot[bot]' }}
    steps:
      - name: Dependabot metadata
        id: dependabot-metadata
        uses: dependabot/fetch-metadata@v1.3.1
      - name: Enable auto-merge for Dependabot PRs
        run: gh pr merge --auto --merge "$PR_URL"
        env:
          PR_URL: ${{ '{{' }}github.event.pull_request.html_url}}
          GITHUB_TOKEN: ${{ '{{' }}secrets.GITHUB_TOKEN}}
      - name: Approve patch and minor updates
        if: ${{ '{{' }}steps.dependabot-metadata.outputs.update-type == 'version-update:semver-patch' || steps.dependabot-metadata.outputs.update-type == 'version-update:semver-minor'}}
        run: gh pr review $PR_URL --approve -b "I'm **approving** this pull request because **it includes a patch or minor update**"
        env:
          PR_URL: ${{ '{{' }}github.event.pull_request.html_url}}
          GITHUB_TOKEN: ${{ '{{' }}secrets.GITHUB_TOKEN}}
      - name: Approve major updates of development dependencies
        if: ${{ '{{' }}steps.dependabot-metadata.outputs.update-type == 'version-update:semver-major' && steps.dependabot-metadata.outputs.dependency-type == 'direct:development'}}
        run: gh pr review $PR_URL --approve -b "I'm **approving** this pull request because **it includes a major update of a dependency used only in development**"
        env:
          PR_URL: ${{ '{{' }}github.event.pull_request.html_url}}
          GITHUB_TOKEN: ${{ '{{' }}secrets.GITHUB_TOKEN}}
      - name: Comment on major updates of non-development dependencies
        if: ${{ '{{' }}steps.dependabot-metadata.outputs.update-type == 'version-update:semver-major' && steps.dependabot-metadata.outputs.dependency-type == 'direct:production'}}
        run: |
          gh pr comment $PR_URL --body "I'm **not approving** this PR because **it includes a major update of a dependency used in production**"
          gh pr edit $PR_URL --add-label "requires-manual-qa"
        env:
          PR_URL: ${{ '{{' }}github.event.pull_request.html_url}}
          GITHUB_TOKEN: ${{ '{{' }}secrets.GITHUB_TOKEN}}
```

The workflow described above has five steps, and it will only execute them for pull requests opened by the user "dependabot[bot]":

1. **Dependabot metadata**: in this step, the workflow gathers information about the pull request that Dependabot opened. The following steps will use this information to decide whether or not it should approve the pull request.
2. **Enable auto-merge for Dependabot PRs**: previously, we had to click on a button to enable auto-merge for a specific pull request. In this step, the workflow will do it for us by running the command `gh pr merge --auto --merge "$PR_URL"`.
3. **Approve patch and minor updates**: this step will check if the pull request is updating a patch or minor version of the dependency and approve it if that is the case. Note that we are using the information collected by step 1 in this step when we call `steps.dependabot-metadata.outputs.update-type`.
4. **Approve major updates of development dependencies**: unfortunately, GitHub workflows don't support if's and else's in the step definition, so we have to create multiple steps to check for different conditions. In this step, the workflow will approve the pull request if it updates a major version of a dependency that is only used in the development or testing environments.
5. **Comment on major updates of non-development dependencies**: this step will leave a comment in the pull request explaining why it didn't approve it. It will also add a label to the pull request to make it easy to track them down. Don't forget that the label used in this step must exist in the repository already, or this command will fail.

Let's push this workflow to main and trigger a new Dependabot pull request as we did at the end of step 1.

Chances are that you will not have the opportunity to see the pull request open because as soon as Dependabot opens it, the new workflow will kick in and approve it, given that it is not a major dependency used in production.

We have then to go to the closed pull requests to find it.

# Step 4: Preventing merging pull requests with failing build

In this step, we will go back to the branch protection rule we created for the main branch and edit it to include a clause that will only merge the pull request after it has a green build.

For this, we go to Settings, Branches, and edit the branch protection rule we created in step 2. On the edit page, we will check the box "Require status checks to pass before merging". Then, in the field "Search for status checks in the last week for this repository", we will select our build check and the review-dependabot-pr and click save:

![Require check status before merge](/assets/img/posts/2022-07-23-automating-dependency-updates-with-dependabot-github-auto-merge-and-github-actions/require-check-status-before-merge.png)

To test this new setting, we can downgrade a dependency and add a test that fails if it gets updated.

In a Rails project, for example, we could downgrade the version for the framework and add a test like the following:

```ruby
# test/integration/rails_version_test.rb

class RailsVersionTest < ActionDispatch::IntegrationTest
  test "version is < 7.0.3" do
    assert Gem::Version.new(Rails.version) < Gem::Version.new("7.0.3")
  end
end
```

If we commit this test with a downgrade of Rails to v7.0.2 and trigger a new Dependabot scan, we should get a new pull request open with a failing build.

The GitHub action dependabot-reviewer will approve the pull request because it is a patch update, but because of the failing build, GitHub won't auto-merge it.

This step finishes the setup of the workflow just as described in the flowchart introduced previously.

## Conclusion

I have to be honest. I only played around with this workflow in a hypothetical project, but I can't think of a strong reason why it wouldn't work on an actual setup.

Also, the workflow should be pretty flexible to fit different needs. For example, we can change Dependabot's configuration to ignore specific dependencies we prefer to update manually.

I hope to use this workflow on a future project to streamline and automate the process of keeping my dependencies up to date.

To have a practical example of how this workflow looks like in a Rails project, one can refer to the repository at [github.com/nicolasiensen/dependabot-test-2](https://github.com/nicolasiensen/dependabot-test-2/pull/8).
