# Aborting irrelevant workflows

All developers have at some point made a commit and then immediately realized that there was a small mistake somewhere that could be fixed in 5 seconds. When we do this quick fix we suddenly have 2 pipelines running, once of which is completely irrelevant.

We want to abort these irrelevant pipelines not necessarily to optimize for speed, but rather to optimize for resources. Less computing time will directly translate into less [money spent for an enterprise](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions). There is more than just speed when you are building a **lean, mean CI machine**

# Practical Part

There is a property called [concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency) in GitHub Actions, which can be used for this exact purpose.

Right before the `jobs` add this piece of configuration,

```yaml
...
  concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: true
...
```
We specify a group using the workflow and the ref to make sure that push events on different branches do not cancel each other out.

Commit your changes to the repository, and trigger a `workflow_dispatch` event from the GitHub UI. The first workflow should have been canceled in favor of the triggered workflow.

`ci.yml` should look something like this now
```yaml
# ci.yml
name: CI Pipeline

on: [pull_request, push, workflow_dispatch]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test_n_lint:
    name: Test n' Lint
    runs-on: ubuntu-latest
    steps:
      # Checkout source code and setup node version.
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v2
        with:
          node-version: "16.x"

      # Cache dependencies
      - name: Cache Yarn
        uses: actions/cache@v3
        id: cache-yarn
        env:
          cache-name: yarn-cache
        with:
          path: |
            node_modules
            .yarn
          key: ${{ runner.os }}-${{ env.cache-name }}-${{ hashFiles('yarn.lock') }}
          restore-keys: ${{ runner.os }}-${{ env.cache-name }}-

      # Install dependencies 

      - name: Install dependencies
        if: steps.cache-yarn.outputs.cache-hit != 'true'
        run: yarn install --prefer-offline --cache-folder .yarn
      
      # Test n' Lint !

      - name: Lint
        run: yarn lint


      - name: Find reference commit on current branch
        if: github.event_name != 'pull_request'
        uses: nrwl/last-successful-commit-action@v1
        id: last_successful_commit
        with:
          branch: ${{ github.ref_name }}
          workflow_id: ci.yml
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup test flags
        run: |
          if [[ "${{ github.event_name }}" == "pull_request" ]];
          then
            echo "TEST_FLAGS=--changedSince ${{ github.event.pull_request.base.sha }}" >> $GITHUB_ENV
          elif [[ -n "${{ steps.last_successful_commit.outputs.commit_hash }}" ]];
          then
            echo "TEST_FLAGS=--changedSince ${{ steps.last_successful_commit.outputs.commit_hash }}" >> $GITHUB_ENV
          fi

      - name: Test
        run: yarn test $TEST_FLAGS
```
