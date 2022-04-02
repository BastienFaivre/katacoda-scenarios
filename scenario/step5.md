# setup-node cache functionality

By looking at the detailed execution of the worlflow, you probably noticed that the main problem that slows down it is the command:

```
yarn install
```

That installs all the necessary packages. Taking into account that the majority of the packages are used at each execution of the workflow, the idea of caching them between the different executions would seem to be a good idea.

# Practical part

By reading the [setup-node](https://github.com/actions/setup-node) functionalities, you can notice that it provides an [action](https://github.com/actions/setup-node#caching-packages-dependencies) that caches the package dependencies. Let's implement it.

Two more parameters need to be added for setup-node in order to use the cache feature:

1. Firstly, we need to specify the package manager used for the project, `yarn` in our case. This is done using the line:
   
   ```
   cache: `yarn`
   ```
2. Secondly, the path to the lock file (`yarn.lock`) should also be specified using:
   
   ```
   cache-dependency-path: ./yarn.lock
   ```

Therefore the setup-node part of the workflow becomes:
```
  ...
- uses: actions/setup-node@v2
  with:
    node-version: '16.x'
    cache: 'yarn'
    cache-dependency-path: ./yarn.lock
  ...
```

Commit and push the changes. Now, in addition to the speed up in execution time, you can notice that the installation of packages is much faster.

Note: to notice the changes, you must run the workflow at least two times, the first one to cache the packages and then in the second run you can see the use of the previous cache.