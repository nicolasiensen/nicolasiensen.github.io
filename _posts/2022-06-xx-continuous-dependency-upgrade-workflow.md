Updating dependencies is part of every longlive software project

There are three players involved in automating this workflow

- **dependabot** opens a pull request when it finds a new version for a dependency
- **dependabot reviewer** is a custom GitHub action that will approve the pull request as long as the dependency update is safe for merging
- **Pull request auto-merge** will automatically merge a dependabot pull request whenever the tests are passing and it has an approval from the dependabot reviewer




With more than ten years



- Trigger the dependabot check manually: Insights -> Dependency graph -> Dependabot -> Click on the latest update (Gemfile) -> Check for updates
- Continuous dependency upgrade flowchart: https://dreampuf.github.io/GraphvizOnline/#digraph%20G%20%7B%0A%20%20%20%20dependabot_creates_new_pr%5Blabel%3D%22PR%20created%20by%20dependabot%22%20shape%3D%22rect%22%5D%0A%20%20%20%20is_build_passing%5Blabel%3D%22Is%20build%5Cnpassing%3F%22%20shape%3D%22diamond%22%5D%0A%20%20%20%20is_major_update%5Blabel%3D%22Is%20major%5Cnupdate%3F%22%20shape%3D%22diamond%22%5D%0A%20%20%20%20merge_pr%5Blabel%3D%22Merge%20PR%22%5D%0A%20%20%20%20is_production_dependency%5Blabel%3D%22Is%20production%5Cndependency%3F%22%20shape%3D%22diamond%22%5D%0A%20%20%20%20pr_is_waiting_for_fix%5Blabel%3D%22PR%20is%20waiting%5Cnfor%20a%20fix%22%20shape%3D%22rect%22%5D%0A%20%20%20%20waiting_for_qa%5Blabel%3D%22Waiting%20for%20QA%22%20shape%3D%22rect%22%5D%0A%20%20%20%20fix_build%5Blabel%3D%22Fix%20the%20build%22%5D%0A%20%20%20%20test_pr%5Blabel%3D%22Test%20PR%22%5D%0A%20%20%20%20%0A%09dependabot_creates_new_pr%20-%3E%20is_build_passing%0A%09is_build_passing%20-%3E%20is_major_update%5Blabel%3D%22Yes%22%5D%0A%20%20%20%20is_build_passing%20-%3E%20pr_is_waiting_for_fix%5Blabel%3D%22No%22%5D%0A%20%20%20%20is_major_update%20-%3E%20merge_pr%5Blabel%3D%22No%22%5D%0A%20%20%20%20is_major_update%20-%3E%20is_production_dependency%5Blabel%3D%22Yes%22%5D%0A%20%20%20%20is_production_dependency%20-%3E%20merge_pr%5Blabel%3D%22No%22%5D%0A%20%20%20%20is_production_dependency%20-%3E%20waiting_for_qa%5Blabel%3D%22Yes%22%5D%0A%20%20%20%20pr_is_waiting_for_fix%20-%3E%20fix_build%0A%20%20%20%20fix_build%20-%3E%20is_major_update%0A%20%20%20%20waiting_for_qa%20-%3E%20test_pr%09%0A%20%20%20%20test_pr%20-%3E%20merge_pr%0A%7D

- Fecth dependabot metadata action: https://github.com/dependabot/fetch-metadata

## Workflow coordination
- Add wait step before running a workflow: https://github.com/marketplace/actions/wait-on-check
- Using workflow_run to wait for a workflow: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#using-data-from-the-triggering-workflow

- Offical Github tutorial to automate dependency upgrade with dependabot: https://docs.github.com/en/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions

## Similar posts
- https://medium.com/@digilist/automatic-dependency-updates-891589688adc
- https://www.nearform.com/blog/github-dependabot-automation/
- https://www.jdno.dev/fully-automated-dependency-upgrades-with-dependabot-and-github-actions/

Passes CI, Minor: https://github.com/nicolasiensen/dependabot-test/pull/8
Passes CI, Major dev:

CI     | Semver | Type | PR
-------|--------|------|---
Passes | Minor  | Prod | https://github.com/nicolasiensen/dependabot-test/pull/10
Passes | Major  | Dev  | https://github.com/nicolasiensen/dependabot-test/pull/9
Passes | Major  | Prod | https://github.com/nicolasiensen/dependabot-test/pull/7
Fails  | Minor  | Dev  | https://github.com/nicolasiensen/dependabot-test/pull/11

