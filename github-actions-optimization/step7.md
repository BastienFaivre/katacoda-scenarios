# Run tests conditionally

In CI you want to run all your tests on each commit, which is a great mechanism for detecting when a change breaks your code base. However, usually all tests are not affected by every commit. So a strategy that would be viable is to only run the tests that are affected by the code change. It is fairly difficult to resolve the affected tests from the changed files, luckily `jest` provides some useful flags that we can use in this regard,

- `--changedFilesWithAncestor` runs the tests that has been affected by the changes in the previous commit. 
- `--changedSince` takes a commit hash as an input and runs the affected changes since the provided commit.

We perhaps would want the following behavior,
- on `push` events we want to run the tests that were affected by the commit
- on `pull_requests` events we want to run all the test that were affected all commits in the pull pull requests

Great! It seems like we can make use of the `--changedFilesWithAncestor` and `--changedSince` respectively to get this desired behavior. However, there is a "hidden" trap with this approach. Let's imagine this scenario,

You introduced a bug in commit 1 that breaks the test suite, the pipeline runs the correct tests and catches the bug. So far everything seems to work correctly. However, in the next commit you do not introduce a fix, or you work on some other file that is not directly affecting the failed test. If we run `jest` with `--changedFilesWithAncestor` it will not run the failing test, and the pipeline may succeed and we have now "hidden" a bug in our code that should have been caught. 

To solve this issue we effectively want to run all tests that are changed since the **previous successful commit**. In that way we can avoid the above scenario.

# Practical Part

To achieve the desired behavior we will use a action called `nrwl/last-successful-commit-action@v1`, which does exactly what the name implies.

First off, we need to add this piece of configuration to `actions/checkout@v2`,

```yaml
...
  - uses: actions/checkout@v2
    with:
      fetch-depth: 0
...
```

This tells the checkout action that we want the *entire* git history, not just the most previous commit. We need this to be able to use the flags provided by `jest` as it relies of the git-history to determine which tests to run.

We will add the following configuration,

```yaml

...
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
...
```

In the first block we set it using the `nrwl/last-successfull-commit-action` action. This action requires three inputs; the branch, the workflow that it needs to check, and a GitHub token which it uses to make some API calls. All actions have access to a GitHub token with limited privileges using `secrets.GITHUB_TOKEN`, you can read more about it [here](https://docs.github.com/en/actions/security-guides/automatic-token-authentication).

In the second block we set the flags depending on if we are dealing with a `pull_request` event or not. We have to make use of some bash scripting here in order to set a environment variable conditionally. Notice that we have a `elif` statement opposed to a `else` statement. We do this to check if there is a commit hash output from the previous step. For instance, on the first commit of a new branch there will be no commit to reference on said branch. In this case we simply run the entire test suite, instead of trying to resolve a successful commit on the base branch.

We now need to run our tests to with the flags, so we change the configuration to this

```yaml
...
  - name: Test
    run: yarn test $TEST_FLAGS
...
```

Commit and push your changes to the repository. If you inspect the workflow logs you should see that none of the tests were run, as we did not change any of the source code.

Now we should test that a change in the implementation should trigger the tests to run. Make a new branch, and make some insignificant change to the `README.md` and commit the changes. If you inspect the logs once again you will see that all of the tests should have run, as it is the first commit of a new branch.

In the file `mult.js` there is an iterative implementation of multiplication using addition. Let's instead use the built-in operators to perform multiplication, change the function `mult(a,b)` to look something like this,

```js
function mult(a,b) {
  return a * b
}
```

Commit and push the changes. This should trigger the tests for **only** the `mult` function to run. Go to the repository and make a pull request to the main branch. This will trigger an additional workflow run, which also only runs the test for the `mult` function. If you look at the commit hash that was used as a reference it should be for the commit where we initially branched off. 

Great! This kind of test running can be a huge time saver for very large test suites, but you will have to be careful to make sure that you are not missing test cases. Let's continue to the next step.

> OBS:
> Switch back to the main branch for future steps!

`ci.yml` should look somethin like this now

```yaml
# ci.yml
name: CI Pipeline

on: [pull_request, push, workflow_dispatch]

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
