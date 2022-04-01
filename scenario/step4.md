# NPM vs Yarn

Let's talk about the choice of the package manager. The two most popular are `npm` and `yarn`.

The most interesting difference considered for this tutorial is the package installation speed. Indeed, `yarn` installs packages in parallel while `npm` installs them sequentially. Therefore this feature should help you to speed your workflows.

Note: see the [resources](#resources) for further information about the difference between the two package managers.

# Practical part

The project currently use `npm` as package manager. To change to `yarn`, do the following steps:

1. Delete the file `package-lock.json` and the folder `node_modules` using the following command:
   
   ```
   rm -rf package-lock.json node_modules
   ```
2. Run the following command to re-install all the packages using `yarn`:

    ```
    yarn install
    ```

    This command will create a new file called `yarn.lock` that is the equivalence of the file `package-lock.json`, but for `yarn`. The `node_modules` folder will also be re-created.

Commit and push the changes and as for the previous step, you should see an improvement in execution time.

Note: to avoid an unexpected behaviour, run the workflow 2-3 times to get an average value.

## <a id="resources"/>Resources

https://www.whitesourcesoftware.com/free-developer-tools/blog/npm-vs-yarn-which-should-you-choose/

https://www.positronx.io/yarn-vs-npm-best-package-manager/

https://www.section.io/engineering-education/npm-vs-yarn-which-one-to-choose/