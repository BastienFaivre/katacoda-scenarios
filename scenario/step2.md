# Presentation

This repository is a simple nodeJS project with TODO...

# The Initial Workflow

The initial workflow is located in `.github/workflows/ci.yml`. Here is the description on this initial version:

- Line 1: The name of the workflow
- Lines 3-7: The events on which the workflow should be triggered
- Line 11: Define the jobs
- Line 12: The name of the unique job
- Line 13: The runner on which the workflow is run
- Line 14: Define the steps
- Line 15: This [action](https://github.com/actions/checkout) allow us to access the repository
- Line 16: A [GitHub Action](https://github.com/actions/setup-node) for node
- Line 17-18: Define the node version
- Line 19: First command, install all the packages
- Line 20: Second command, run the tests

To summarize, this initial workflow simply uses node version 16.x on a macOS machine, install all packages and run all the tests.