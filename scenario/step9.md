Time improvements are really interesting. However, there exist other improvements unrelated to time that indirectly save you time.

# Use Build Matrix

Let's say that the project should be now tested, not only on `ubuntu-latest` (i.e Ubuntu 20.04), but also on `macOS-latest` and `ubuntu-18.04`. Furthermore, the project should also be working for older version of nodeJS such as `14` or even `12`.

A first idea to implement this is to copy the whole `test_n_lint` job for all possible pairs [runner, node version]:

| runner | node version |
|----|--------------|
|ubuntu-latest|12|
|ubuntu-latest|14|
|ubuntu-latest|16|
|ubuntu-18.04|12|
|ubuntu-18.04|14|
|ubuntu-18.04|16|
|macOS-latest|12|
|macOS-latest|14|
|macOS-latest|16|

This will lead to 9 almost *copy jobs* with just two parameters changing each time. Furthermore, the code lenght is clearly increased. Isn't there an easier way to implement this ? The answer is yes, using [build matrix](https://docs.github.com/en/actions/using-jobs/using-a-build-matrix-for-your-jobs).

# Practical Part

A [matrix](https://docs.github.com/en/actions/using-jobs/using-a-build-matrix-for-your-jobs) allows us to define multiple similar jobs using a single job definition. The idea is to use a matrix to define all variables that need to be changed in the job. In our case, the runner and the node version. This id done by defining a strategy matrix in the job definition:

```
strategy:
  matrix:
    runner: [ubuntu-latest, ubuntu-18.04, macOS-latest]
    node: [12, 14, 16]
```

And then retrieve the current value of the variables using `${{ matrix.runner }}` and `${{ matrix.node }}`. The final workflow is therefore:

```
TODO add workflow
```

Note that the matrix feature also provides tools to [include](https://docs.github.com/en/actions/using-jobs/using-a-build-matrix-for-your-jobs) and [exclude](https://docs.github.com/en/actions/using-jobs/using-a-build-matrix-for-your-jobs) particular sets of variables if necessary.

Commit and push these changes. Here, you will not see any time improvement since we just added 8 more jobs compared to the previous version. However, the idea with this improvement is to show that a clean and intelligent code makes things faster to understand. Let's say that a new employee just arrived in your company and need to understand the workflows. Understanding them will be faster since they are defined in a proper way. Just 4 lines were added against 8 copies of the initial job with the first idea!