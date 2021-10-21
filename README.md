# ![wemogy](https://wemogyimages.blob.core.windows.net/logos/wemogy-github-tiny.png) Get Environment (GitHub Action)

A GitHub Action to detect on which branch a workflow is running on and derive an environment name from that.

## Usage

```yaml
- name: Get Environment
  id: get-environment
  uses: wemogy/get-environment-action@1.2.2
  
- run: echo ${{ steps.get-environment.outputs.env }}
```

## Inputs

| Input     | Description                                                                       |
| --------- | --------------------------------------------------------------------------------- |
| `prod-branch` | Name of the branch that should result in environment 'staging' (Default: staging) |
| `prod`    | Name of the branch that should result in environment 'prod' (Default: prod)       |
| `dev`     | Name of the branch that should result in environment 'dev' (Default: main)        |
| `default` | Default environment, if no branch match found (Default: dev)                      |

## Outputs

| Output         | Description                                                    |
| -------------- | -------------------------------------------------------------- |
| `env`          | Does the current branch exactly match one of the environments? |
| `exact-match`  | The current branch exactly matches                             |
| `pull-request` | Is the current branch part of a pull request?                  |
| `branch-name`  | Name of the branch this is running on.                         |



name: Get Environment (wemogy)
author: wemogy
description: Detects on which branch a workflow is running on and derives an environment name from that
branding:
  icon: "git-branch"
  color: "blue"

inputs:
  prod-branch:
    description: "Name of the branch that should result in environment 'prod'"
    default: "prod"
    required: true
  prod-domain-prefix:
    description: "Prefix to use in domains when environment is 'prod'"
    default: ""
    required: true
    
  staging-branch:
    description: "Name of the branch that should result in environment 'staging'"
    default: "staging"
    required: true
  staging-domain-prefix:
    description: "Prefix to use in domains when environment is 'staging'"
    default: "staging"
    required: true
    
  dev-branch:
    description: "Name of the branch that should result in environment 'dev'"
    default: "main"
    required: true
  dev-domain-prefix:
    description: "Prefix to use in domains when environment is 'dev'"
    default: "dev"
    required: true
  
  pr-environment:
    description: "Default environment, if no branch match found"
    default: "dev"
    required: true

outputs:
  env:
    description: "The Environment this branch is targeting"
    value: ${{ steps.check.outputs.env }}
  exact-match:
    description: "Does the current branch exactly match one of the environments?"
    value: ${{ steps.check.outputs.exact-match }}
  pull-request:
    description: "Is the current branch part of a pull request?"
    value: ${{ steps.check.outputs.pull-request }}
  branch-name:
    description: "The name of the branch this is running on"
    value: ${{ steps.check.outputs.branch-name }}
  domain-prefix:
    description: "The name of the branch this is running on"
    value: ${{ steps.check.outputs.domain-prefix }}
  version-suffix:
    description: "The name of the branch this is running on"
    value: ${{ steps.check.outputs.version-suffix }}

runs:
  using: "composite"
  steps:
    - name: Commit and push
      id: check
      shell: bash
      run: |
        # Find current branch
        # When executing this in context of a pull request, the GITHUB_HEAD_REF points to the source branch.
        # When executing this outside of a pull request, the GITHUB_HEAD_REF is empty and GITHUB_REF points to the current branch.
        if [[ "${GITHUB_HEAD_REF}" != "" ]]; then
          BRANCH_NAME=$(echo ${GITHUB_HEAD_REF})
          IS_PULL_REQUEST=true
        else
          BRANCH_NAME=$(echo ${GITHUB_REF#refs/heads/})
          IS_PULL_REQUEST=false
        fi

        echo "Current branch: $BRANCH_NAME"
        echo "::set-output name=branch-name::$BRANCH_NAME"
        echo "::set-output name=pull-request::$IS_PULL_REQUEST"

        # Compare current branch with environments
        if [[ $BRANCH_NAME == "${{ inputs.staging }}" ]]; then
          echo "Environment: staging"
          echo "::set-output name=env::staging"
          echo "::set-output name=exact-match::true"
          echo "::set-output name=domain-prefix::${{ staging-domain-prefix }}"
          echo "::set-output name=version-suffix::staging"
        elif [[ $BRANCH_NAME == "${{ inputs.prod }}" ]]; then
          echo "Environment: prod"
          echo "::set-output name=env::prod"
          echo "::set-output name=exact-match::true"
          echo "::set-output name=domain-prefix::${{ prod-domain-prefix }}"
          echo "::set-output name=version-suffix::prod"
        elif [[ $BRANCH_NAME == "${{ inputs.dev }}" ]]; then
          echo "Environment: dev"
          echo "::set-output name=env::dev"
          echo "::set-output name=exact-match::true"
          echo "::set-output name=domain-prefix::${{ dev-domain-prefix }}"
          echo "::set-output name=version-suffix::dev"
        elif [[ $IS_PULL_REQUEST ]]; then
          echo "Pull Request detected. Environment: ${{ inputs.pr-environment }}"
          echo "::set-output name=env::${{ inputs.pr-environment }}"
          echo "::set-output name=exact-match::false"
          echo "::set-output name=domain-prefix::pr-${{ github.event.pull_request.number }}"
          echo "::set-output name=version-suffix::pr-${{ github.event.pull_request.number }}"
        else
          echo "::warning title=Could not find environment::No matching branch and no Pull Request found."
        fi
