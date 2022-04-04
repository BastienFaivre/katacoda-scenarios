# Running independent jobs in parallel

So far we have run all our different steps in sequence in the large `Test n' Lint` job. However, the actual `Test` and `Lint` are completely independent of each other but they both depend on the `Install dependencies` step. So what we want to do is to first run the dependency installation, and then run both the `Test` and `Lint` steps in parallel

# Practical Part

By default, GitHub Action runs each job in parallel on separate virtual machines. We can also specify a dependency relationship between jobs, so that one jobs waits for another to finish before it starts.

As the different jobs are run on different machines, there is no "nice" way of transferring data between jobs. We will simply have to rely on the caching mechanism we built get the installed dependencies from the `Install dependencies` step.

We split up the `Test 'n Lint` job into three jobs; `Install Dependencies`, `Test` and `Lint`.

The `Install Dependecies job` will look something like this,

```yarn
...
  install_dependencies:
    name: Install Dependencies
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2 # we can once again do a shallow clone
      - uses: actions/setup-node@v2
        with:
          node-version: '16.x'

      - name: Cache Yarn
        id: cache-yarn
        uses: actions/cache@v3
        env:
          cache-name: yarn-cache
        with:
          path: |
            node_modules
            .yarn
          key: ${{ runner.os }}-${{ env.cache-name }}-${{ hashFiles('**/yarn.lock') }}
          restore-keys: ${{ runner.os }}-${{ env.cache-name }}-

      - name: Install dependencies
        if: steps.cache-yarn.outputs.cache-hit != 'true'
        run: yarn install --prefer-offline --cache-folder .yarn
...
```

The `Lint` job will look like this,

```yarn
...
  lint:
    name: Lint
    needs: install_dependencies # specify dependency relationship
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2 # we can once again do a shallow clone
      - uses: actions/setup-node@v2
        with:
          node-version: '16.x'

      # Get the cache saved by install_dependencies
      - uses: actions/cache@v3
        env:
          cache-name: yarn-cache
        with:
          path: |
            node_modules
            .yarn
          key: ${{ runner.os }}-${{ env.cache-name }}-${{ hashFiles('**/yarn.lock') }}


      - name: Lint
        run: yarn lint "./**/*.js"
...
```

And finally, the `Test` job will look like this,

```yaml
...
  test:
    name: Test
    runs-on: ubuntu-latest
    needs: install_deps
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - uses: actions/cache@v3
        env:
          cache-name: yarn-cache
        with:
          path: |
            node_modules
            .yarn
          key: ${{ runner.os }}-${{ env.cache-name }}-${{ hashFiles('**/yarn.lock') }}


      - name: Find reference commit on current branch
        if: github.event_name != 'pull_request'
        uses: nrwl/last-successful-commit-action@v1
        id: last_successful_commit
        with:
          branch: '${{ github.ref_name }}'
          workflow_id: 'ci.yml'
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
...
```

Commit and push the changes. If you inspect the workflow in the repository, you will see the dependency relationship visualised, and you will see that the `Test` and `Lint` jobs are run in parallel.

Note that it is very likely that is parallelisation will not improve the overall speed of the pipeline for this project, due to its small scale. This is due to the overhead that we introduced where we need to checkout the code, setup node, and fetch data from the cache for each job. Not to mention that we need to wait for the GitHub Action runner to find three virtual machines, which it self can take up to 15 seconds. Essentially we introduce about 25 - 35 seconds of overhead to save less then one second on the `Lint` job.

But let's imagine if you had 2 independent jobs that took 10 minutes each, then this overhead would be negligible as you would save almost 10 minutes of the entire pipeline which would be a huge performance gain.
