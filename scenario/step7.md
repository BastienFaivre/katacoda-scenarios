# Run tests conditionally

In CI you want to run all your tests on each commit, which is a great mechanism for detecting when a change breaks your code base. However, usually all tests are not affected by every commit. So a strategy that would be viable is to only run the tests that are affected by the code change. It is fairly difficult to resolve the affected tests from the changed files, luckily `jest` provides some useful flags that we can use in this regard,

- `--changedFilesWithAncestor` runs the tests that has been affected by the changes in the previous commit. 
- `--changedSince` takes a commit hash as an input and runs the affected changes since the provided commit.

We perhaps would want the following behavior,
- on `push` events we want to run the tests that was affected by the commit
- on `pull_requests` events we want to run all the test that was affected all commits in the pull pull requests

Great! It seems like we can make use of the `--changedFilesWithAncestor` and `--changedSince` respectively to get this desired behavior. However, there is a "hidden" trap with this approach. Let's imagine this scenario,

You introduced a bug in commit 1 that breaks the test suite, the pipeline runs the correct tests and catches the bug. So far everything is seems to work correctly. However, in the next commit you do not introduce a fix, or you work on some other file that is not directly affecting the failed test. If we run `jest` with `--changedFilesWithAncestor` it will not run the failing test, and the pipeline may succeed and we have now "hidden" a bug in our code that should have caught. 

To solve this issue we effectively want to run all tests that are changed **previous successful commit**. In that way we can avoid the above scenario.

# Practical Part

To achieve the desired behavior we will use a action called `nrwl/last-successfull-commit-action@v1`, which does exactly what the name implies.

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

  - name: Find refernce commit on current branch
    if: github.event_name != "pull_request"
    uses: nrwl/last-successfull-commit-action@v1
    id: last_successful_commit
    with:
      branch: $${{ github.ref_name }}
      workflow_id: ci.yml
      github_token: $${{ secrets.GITHUB_TOKEN }}
    env:
      REFERENCE_COMMIT: $${{ steps.last_sucessful_commit.outputs.commit_hash }}

  - name: Find reference commit for pull request
    if: github.event_name == 'pull_request'
    run: echo "REFERENCE_COMMIT=${{ github.event-pull_request.base.sha }}" >> $GITHUB_ENV

  - name: Setup test flags
    run: |
      if [[ -n "$REFERENCE_COMMIT" ]]; then
        echo "TEST_FLAGS=--changedSince $REFERENCE_COMMIT" >> $GITHUB_ENV
      fi
```

We have two mutually exclusive steps, one that is run on `pull_request` events, and one that is run on all other events (`push` and `workflow_dispatch`). Our goal is to set an environment variable called `REFERENCE_COMMIT` to the desired commit hash.

In the first block we set it using the `nrwl/last-successfull-commit-action` action. This action requires three inputs; the branch, the workflow that it needs to check, and a GitHub token which it uses to make some API calls. All actions have access to a GitHub token with limited privileges using `secrets.GITHUB_TOKEN`, you can read more about it [here](TODO).

The second block simply set the environment variable to the base of the branch of the pull request.

The third block is a safe guard to ensure that we have resolved a reference commit, as in some cases we may not. For instance, the first commit on a new branch will not be able to resolve using `nrwl/last-successfull-commit-action`, so in that case we will run the entire test suite.

Observe that we use 2 different syntaxes for setting the environment variables in this snippet. GitHub Actions requires you to have either a `with` label or a `run` label but not both. To solve this we can use the hack of using echoing into `GITHUB_ENV` to set the environment variables. Also, this is the only way to set a environment variables conditionally.

We now need to run our tests to with the flags, so we change the configuration to this

```yaml
...
  - name: Test
    run: yarn test $TEST_FLAGS
...
```

Commit and push your changes to the repository. If you inspect the workflow logs you should see that none of the tests were run, as we did not change any of the source code.

Make a new branch, and make some insignificant change to the `README.md` and commit the changes. If you inspect the logs once again you will see that all of the test should have run.

Now we should test that a change in the implementation should trigger the test to run.

In the file `mult.js` there is an iterative implementation of multiplication using addition. Let's instead use the built-in operators to perform multiplication, change the function `mult(a,b)` to look something like this,

```js
function mult(a,b) {
  return a * b
}
```

Commit and push the changes. This should trigger the tests for **only** the `mult` function to run. Go to the repository and make the repository a pull request to the main branch. This will trigger an additional workflow run, which also only runs the test for the `mult` function. If you look at the commit hash that was used as a reference it should be for the commit where we initially branched off.

Great! This kind of test running can be a huge time saver for very large test suites, but you will have to be careful to make sure that you are not missing test cases. Let's continue to the next step.
