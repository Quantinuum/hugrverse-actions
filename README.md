# hugrverse-actions

Reusable workflows for projects in the hugrverse.

To use a workflow in your project, add define it in a `.yml` file in your project's
`.github/workflows` directory.
See the workflow list below for usage instructions, including the workflow triggers.

Some workflows may require additional inputs, such as a [`GITHUB_PAT`] to
access the GitHub API. Follow github's instructions for [generating fine-grained access
tokens](https://github.com/settings/personal-access-tokens/new), and store it in your
repository secrets.
For Quantinuum projects, ask the `HUGR` team for a [@hugrbot](https://github.com/hugrbot) token.

The following workflows are available:

- [`add-to-project`](#add-to-project): Adds new issues to a GitHub project board when they are created.
- [`breaking-change-to-main`](#breaking-change-to-main): Checks for breaking changes targeting a protected branch (by default `main`) and disallows them when a separate release branch is specified.
- [`coverage-trend`](#coverage-trend): Checks the coverage trend for the project, and produces a summary that can be posted to slack.
- [`create-issue`](#create-issue): Creates a new issue in the repository, avoiding duplicates.
- [`drop-cache`](#drop-cache): Drops the cache for a branch when a pull request is closed.
- [`pr-title`](#pr-title): Checks the title of pull requests to ensure they follow the conventional commits format.
- [`py-semver-checks`](#py-semver-checks): Runs `griffe` on a PR against the base branch, and reports back if there are breaking changes.
- [`rs-semver-checks`](#rs-semver-checks): Runs `cargo-semver-checks` on a PR against the base branch, and reports back if there are breaking changes. If you need more customisation, use the homonymous action instead (see [Using the Semver action directly](#using-the-semver-action-directly)).
- [`slack-notifier`](#slack-notifier): Post comments on slack, with a rate limit to avoid spamming the channel.

## [`add-to-project`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/add-to-project.yml)

Adds new issues to a GitHub project board when they are created.

### Usage
```yaml
name: Add issues to project board
on:
  issues:
    types:
      - opened

jobs:
    add-to-project:
        uses: quantinuum/hugrverse-actions/.github/workflows/add-to-project.yml@main
        with:
            project-url: https://github.com/orgs/{your-org}/projects/{project-id}
        secrets:
            GITHUB_PAT: ${{ secrets.ADD_TO_PROJECT_PAT }}
```

### Token Permissions

The fine-grained `GITHUB_PAT` secret must include the following permissions:

| Permission | Access |
| --- | --- |
| Projects | Read and write |
| Pull requests | Read |

The fine-grained access token **must** be defined in the organization that owns the project. This may require a different token from the one used in other workflows.

If the repository is private and the project is in a different organization, it is not possible to define fine-grained access tokens with simultaneous access to both. In those cases, you will need an unrestricted _classical_ github token instead.

## [`breaking-change-to-main`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/breaking-change-to-main.yml)

Checks for breaking changes targeting `protected_branch` (by default `main`) and
disallows them when a separate `release_branch` is specified. This is useful to
prevent accidental breaking changes from being merged into `protected_branch`,
when non-breaking patch releases are still expected to be published before the
next breaking release.

By default, the `release_branch` is set to the `protected_branch`, and in this
case, the workflow will be a no-op.

### Usage
```yaml
name: Check for breaking changes targeting main
on:
  pull_request_target:
    branches:
      - main
    types:
      - opened
      - edited
      - synchronize
      - labeled
      - unlabeled
  merge_group:
    types: [checks_requested]

jobs:
    breaking-change-to-main:
        uses: quantinuum/hugrverse-actions/.github/workflows/breaking-change-to-main.yml@main
        secrets:
            GITHUB_PAT: ${{ secrets.GITHUB_PAT }}
        with:
            # The dedicated release branch, or the main branch if none exists.
            # Typically, this will be a [repo variable](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables).
            # Defaults to the same value as protected_branch.
            release_branch: release-branch
            # The protected branch, typically main (default) or master.
            protected_branch: main
```

### Token Permissions

The fine-grained `GITHUB_PAT` secret must include the following permissions:

| Permission | Access |
| --- | --- |
| Pull requests | Read and write |

## [`coverage-trend`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/coverage-trend.yml)

Compares the project coverage on [Codecov](https://codecov.io/) against the last workflow run,
and produces a summary of the changes that can be posted to slack.

If the project didn't have new commits that changed the coverage since the last run,
the `should_notify` output will be set to `false` and the `msg` output will be empty.

### Usage
```yaml
name: Notify coverage changes
on:
  schedule:
    # 04:00 every Monday
    - cron: '0 4 * * 1'
  workflow_dispatch: {}

jobs:
    coverage-trend:
        uses: quantinuum/hugrverse-actions/.github/workflows/coverage-trend.yml@main
        secrets:
            CODECOV_GET_TOKEN: ${{ secrets.CODECOV_GET_TOKEN }}
    # Post the result somewhere.
    notify-slack:
      needs: coverage-trend
      runs-on: ubuntu-latest
      if: needs.coverage-trend.outputs.should_notify == 'true'
      steps:
        - name: Send notification
          uses: slackapi/slack-github-action@v1.27.0
          with:
            channel-id: "SOME CHANNEL ID"
            slack-message: ${{ needs.coverage-trend.outputs.msg }}
          env:
            SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Outputs

- `should_notify`: Whether there has been a change in coverage since the last run, which we can post about.
- `msg`: A message summarising the coverage changes. This is intended to be posted to slack.

### Token Permissions

`CODECOV_GET_TOKEN` is a token generated by Codecov to access the repository's coverage data.

## [`drop-cache`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/drop-cache.yml)

Drops the cache for a branch when a pull request is closed. This helps to avoid
cache pollution by freeing up some of github's limited cache space.

### Usage
```yaml
name: cleanup caches by a branch
on:
  pull_request:
    types:
      - closed

jobs:
    drop-cache:
        uses: quantinuum/hugrverse-actions/.github/workflows/drop-cache.yml@main
```

## [`create-issue`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/create-issue.yml)

Creates a new issue in the repository, avoiding duplicates.
The workflow takes a "unique-label" input, which is used to check if an issue with that label already exists.

The specified labels must already exist in the repository, otherwise the workflow will fail.

### Usage
```yaml
name: Create an issue
on:
  schedule:
    # 12:00 every Monday
    - cron: '0 12 * * 1'

jobs:
    create-issue:
        uses: quantinuum/hugrverse-actions/.github/workflows/create-issue.yml@main
        secrets:
            GITHUB_PAT: ${{ secrets.GITHUB_PAT }}
        with:
            title: "Hello 🌎!"
            body: "This is a new issue."
            unique-label: "hello-world"
            # Optionally, set the target repository.
            repository: "quantinuum/hugrverse-actions"
            # Optional list of labels to add to the issue.
            other-labels: "greetings,scheduled"
```

### Token Permissions

The fine-grained `GITHUB_PAT` secret must include the following permissions:

| Permission | Access |
| --- | --- |
| Issues | Read and write |


## [`pr-title`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/pr-title.yml)

Checks the title of pull requests to ensure they follow the [conventional
commits](https://www.conventionalcommits.org/en/v1.0.0/) format. If the title
does not follow the conventional commits, a comment is posted on the PR to help
the user fix it.

### Usage
```yaml
name: Check Conventional Commits format
on:
  pull_request_target:
    branches:
      - main
    types:
      - opened
      - edited
      - synchronize
      - labeled
      - unlabeled
  merge_group:
    types: [checks_requested]

jobs:
    check-title:
        uses: quantinuum/hugrverse-actions/.github/workflows/pr-title.yml@main
        secrets:
            GITHUB_PAT: ${{ secrets.GITHUB_PAT }}
```

### Token Permissions

The fine-grained `GITHUB_PAT` secret must include the following permissions:

| Permission | Access |
| --- | --- |
| Pull requests | Read and write |

## [`py-semver-checks`](https://github.com/quantinuum/hugrverse-actions/blob/main/py-semver-checks/action.yml)

Runs [`griffe`](https://mkdocstrings.github.io/griffe/) on a PR against the base branch,
and reports back if there are breaking changes in the Python API.

If the PR has a breaking change, posts a comment with a summary of the changes.

### Usage
```yaml
name: Python Semver Checks
on:
  pull_request_target:
    branches:
      - main

jobs:
  semver-checks:
    name: Python semver-checks 🐍
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_PAT }}
    steps:
      - uses: quantinuum/hugrverse-actions/py-semver-checks@main
        with:
          packages: python-package1 path/to/python-package2
          baseline-rev: main  # If not present, defaults to base branch of the PR
          token: ${{ secrets.GITHUB_PAT }}
```

### Token Permissions

The fine-grained `GITHUB_PAT` secret must include the following permissions:

| Permission | Access |
| --- | --- |
| Pull requests | Read and write |

Note that repository secrets are not available to forked repositories on `pull_request` events.
To run this workflow on pull requests from forks, ensure the action is triggered by a `pull_request_target` event instead.

## [`rs-semver-checks`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/rs-semver-checks.yml)

Runs `cargo-semver-checks` on a PR against the base branch, and reports back if
there are breaking changes.

If the PR has a breaking change, posts a comment with a summary of the changes.

### Usage

```yaml
jobs:
  semver-checks:
    name: Rust semver-checks 🦀
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_PAT }}
    steps:
      - uses: quantinuum/hugrverse-actions/rs-semver-checks@main
        with:
          baseline-rev: main  # If not present, defaults to base branch of the PR
          token: ${{ secrets.GITHUB_PAT }}
```


The workflow compares against the base branch of the PR by default. Use the `baseline-rev` input to specify a different base commit.

### Token Permissions

The fine-grained `GITHUB_PAT` secret must include the following permissions:

| Permission | Access |
| --- | --- |
| Pull requests | Read and write |

Note that repository secrets are not available to forked repositories on `pull_request` events.
To run this workflow on pull requests from forks, ensure the action is triggered by a `pull_request_target` event instead.

## [`slack-notifier`](https://github.com/quantinuum/hugrverse-actions/blob/main/.github/workflows/slack-notifier.yml)

Post comments on slack using
[slackapi/slack-github-action](https://github.com/slackapi/slack-github-action),
adding a rate limit to avoid spamming the channel.

### Usage
```yaml
name: Send a slack message
on:
  pull_request:
    branches:
      - main

jobs:
    message-slack:
        uses: quantinuum/hugrverse-actions/.github/workflows/slack-notifier.yml@main
        with:
            channel-id: "SOME CHANNEL ID"
            slack-message: "Hello 🌎!"
            # A minimum time in minutes to wait before sending another message.
            timeout-minutes: 60
            # A repository variable used to store the last message timestamp.
            timeout-variable: "HELLO_MESSAGE_TIMESTAMP"
        secrets:
            GITHUB_PAT: ${{ secrets.GITHUB_PAT }}
            SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Inputs

- `channel-id`: The ID of the channel to post the message to. (**required**)
- `slack-message`: The message to post. (**required**)
- `timeout-variable`: A repository variable used to store the last message timestamp. (**required**)
- `timeout-minutes`: A minimum time in minutes to wait before sending another message. Defaults to 24 hours.

### Outputs

- `sent`: A boolean indicating if the message was sent.

### Token Permissions

`SLACK_BOT_TOKEN` is a token generated by Slack with `chat:write` access to the
channel. See the
[slackapi/slack-github-action](https://github.com/slackapi/slack-github-action?tab=readme-ov-file#technique-2-slack-app)
documentation for more information.
If you are using a slack app, make sure to add it to the channel.
See formatting options in the [Slack API documentation](https://api.slack.com/reference/surfaces/formatting).

The fine-grained `GITHUB_PAT` secret must include the following permissions:

| Permission | Access |
| --- | --- |
| Variables (repository) | Read and write |
