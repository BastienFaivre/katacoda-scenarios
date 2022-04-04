# Introduction

In this tutorial, you will learn some techniques to speed up the execution time of your GitHub Actions for your nodeJS project. Some other tips will be given to improve your pipelines, not necessarily related to time improvement.

## Structure

You start with a provided sample nodeJS project and a naive GitHub Actions workflow. Then, at each step, one improvement is presented and explained to you. Alongside these explanations, you will modify the workflow code to apply the improvement. Then at the end of the step, you will be able to run the workflow and notice the potential time improvement compared to the previous versions.

## Learning Objective

- Learn the basics of GitHub Actions
- Learn techniques to speed up workflows
- Learn *when* to use these techniques
- Save precious time and resources

## Prerequisite Skills

This tutorial is for anyone who is currently working or want to work on a nodeJS project and use GitHub Actions as CI but don't know how to write proper workflows and optimize them.

## Table of Contents

- Getting started
- Initial Version
- Improvement #1: The Choice of Runner
- Improvement #2: NPM vs Yarn
- Improvement #3: setup-node cache functionality
- Improvement #4: More in-depth caching
- Improvement #5: Run tests conditionally
- Improvement #6: Aborting irrelevant workflows
- Bonus Round: Running independent jobs in parallel 