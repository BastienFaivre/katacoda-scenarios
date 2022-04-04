# Aborting irrelevant workflows

All developers have at some point made a commit and then immediately realized that there was a small mistake somewhere that could be fixed in 5 seconds. When we do this quick fix we suddenly have 2 pipelines running, once of which is completely irrelevant.

We want to abort these irrelevant pipelines not necessarily to optimize for speed, but rather to optimize for resources. The less computing time will directly translate into less [money for an enterprise](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions). There is more than just speed when you are building a **lean, mean CI machine**

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
