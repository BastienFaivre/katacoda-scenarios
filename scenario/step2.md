# The choice of Runner

The choice of the type of machine on which the workflow will run is the first improvement you can achieve. Indeed, depending on the tasks you want to accomplish in your workflow, a specific runner may perform better and therefore faster than another runner.

GitHub Actions can use two types of runners:
- **GitHub-hosted runners**: each job of a workflow will run in a new instance of a virtual environement cleaned and maintained by GitHub. The list of all possible runners can be found [here](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#choosing-github-hosted-runners).
- **Self-hosted runners**: each job of a workflow will run in a runner that you host and therefore can customize to satisfy your needs. This kind of runners could be useful is your jobs need a special software that are not installed on the GitHub-hosted runners for example. More information about this kind of runners can be found [here](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners).

You will find a list of differences between these two types of runners [here](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners#differences-between-github-hosted-and-self-hosted-runners). In the tutorial, you will use the GitHub-hosted runners.

## Practical part

In the workflow file, the runner is specified using the `runs-on` attribute. Currently, the runner is a macOS machine. Let's see if a Linux based machine can perform better.

To do this, change the line 
```
runs-on: macOS
```
With the line
```
runs-on: ubuntu-latest
```
Note: `ubuntu-latest` means Ubuntu 20.04.

Now commit and push this change and you should see a small improvement (few seconds) in execution time.