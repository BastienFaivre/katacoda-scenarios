# More in-depth caching

The caching mechanism of `setup-node` uses another actions that is called `actions/cache` under the hood. If we want more control of our caching mechanism we can use that action directly.

In a project there are usually a lot of caching done in relation to different software used for building and testing the code. For instance, both `jest` and `eslint` maintain caches in order to improve performance. As each run of a workflow is essentially run on a completely new machine, these caches will not persist and therefore not be utilized effectively.

In the case of this very simple project there is too much overhead in caching `jest` and `eslint` caches, but you will have to experiment to test if your project will benefit of having these caches persist across workflow runs.

Like stated in the previous step, it makes sense to cache dependencies, i.e. `node_modules`, to avoid reinstalling the same dependencies repeatedly. This is great, but when you add a new dependency you will invalidate the cache and have to re-install the same dependencies. `yarn` maintains a global package cache that it uses to avoid unnecessary network calls when installing packages.

# Partial Part

First we remove the lines,

```yaml
...
    cache: "yarn"
    cache-dependency-path: ./yarn.lock
...
```

that we added in the previous step to the `setup-node` block. We now add the following configuration.

```yaml
...
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
...
```

Okey, lets break this down.

- `id` specifies a unique identifier for this named step
- in the `env` block we specify a environment variable that represents a name for this cache
- `path` specifies the paths to the resources that we want to cache.
- `key` specifies the key that is used to retrieve the cache.
  - We use a key composed of multiple different pieces of data, as we want to be able to differentiate between different caches.
  - The function `hashFiles` will create a hash from the contents of files, which we then can use in the key.
- `restore-keys` specifies "backup" keys that will be checked if we do not get a cache hit. The secondary checks are partial checks, and that is why have a similar key as a `restore-key`
  - Let's say we have generated a hash key `Linux-yarn-cache-d5542f8c120770a2b15f7e9ad031a6bd2dd0d4a0d8bc3ed24b414a38edca2054`
  - We then add a new package which generates a cache key `Linux-yarn-cache-b56ec76d6c398c6414452554a4b585e4df789f7c797b4dba9865fe61d31371b9`. We will get a cache miss on the `key`, but then the action will do a secondary partial check of the `Linux-yarn-cache-` part and find a match on the previous cache. This enables us to not having to re-install everything if we get a cache miss.

Caches are stored for 7 days, and if you reach the max capacity of storage caches will be evicted based on how often they are used.


`actions/cache@v3` outputs a boolean value that indicates if there was a cache hit or miss. We can use this output to skip the dependency installation step completely if we have a cache hit. We can accesses the output value using the `id` we defined for the cache step. Edit the `Install dependencies` step to look like this,


```yaml
...
  - name: Install dependencies
  if: steps.cache-yarn.outputs.cache-hit != 'true'
  run: yarn install --prefer-offline --cache-folder .yarn
...
```

We added the `--cache-folder` flag to specify where the global `yarn` cache is kept, and the `--prefer-offline` flag to tell `yarn` to first look locally for packages before it tries to grab them from the internet.


Commit your changes, this will initially fetch all packages as we have changed our caching system, note the installation time of `yarn install`. Once the pipeline is run you can trigger a `workflow_dispatch` event from the GitHub UI to see the caching in action. 

Add a new package to the project with `yarn add` and commit the changes. This should invalidate the cache and cause a new `yarn install`, but the install time should be significantly faster.
