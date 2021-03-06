# Publishing Modules

- Applicable to both public and private Forges.

## Updating the Module Metadata

- Make sure the following files have been created in your module, and are up to date.
  - These are used by Puppet Forge to create details for your module's users.
  - `README.md`
  - `CHANGELOG.md`
  - `metadata.json`
  - `LICENSE`
- Keyword tags in `metadata.json` are used by Puppet Forge's search engine to help users find a module.
- Tags cannot contain spaces, and should not match and valid value from `$facts['operatingsystem']`.

  ```json
  "tags": [
      "agent",
      "server"
  ]
  ```

## Packaging a Module

- After verifying everything is crrect, use `puppet module build` to package the module.
- The `pkg/` directory can become very full over time.
  - It is recommended to add this directory to `.gitignore`.

## Uploading a Module to the Puppet Forge

- No API available.
- In the web console:
  - Click Publish.
  - Select the module package you created with `puppet module build`.
  - Upload.
- The results of standard tests and community feedback will be added to the page as they become available.

## Publishing a Module on GitHub

- Most common location to accept bug reports and pull requests.
- Install Git.

  ```bash
  sudo yum install -y git
  ```

- Configure the Git software with your username and email address.

  ```bash
  git config --global user.name "Jane Doe"
  git config --global user.email "janedoe@email.com"
  ```

- Create a GitHub.com account.
- Create a new repository in GitHub.com, named after the module prefixed with "puppet-".
- Set up version tracking in your module.

  ```bash
  cd [MODULE DIRECTORY]
  git init
  ```

- Add any files to avoid uploading to the source repository to `.gitignore`.
  - Binary packages of the module.
  - Dependency fixtures created by rspec for testing.

    **MODULE DIRECTORY/.gitignore**

    ```ini
    # .gitignore
    /pkg/
    /spec/fixtures/
    ```

- Commit your module to the Git repository.

  ```bash
  git add --all
  git commit -m "Initial Commit"
  ```

- Configure GitHub.com as the remote origin.

  ```bash
  git remote add origin https://github.com/[username]/[reponame].git
  git push -u origin master
  ```

- Whenever you publish a change to the module, make sure to update the version number and changes for the version in the following files.
  - `metadata.json`
  - `README.md`
  - `CHANGELOG.md`
  - Then, commit these changes and push to GitHub.

    ```bash
    git commit -a -m "Updated documentation for version X.Y"
    git push -u origin master
    ```

## Automating Module Publishing

- There is a community-provided Ruby gem that automates the task of updating your module on the Forge.
  - maestrodev/puppet-blacksmith
  - Requires adding changes to Rakefile, and placing your Puppet Forge credentials in a text file in your home directory.

    ```bash
    rake module:bump:patch
    # Edit CHANGELOG.md
    rake module:tag
    rake module:push
    ```

  - You can also use `module:bump:minor` for new features, or `module:bump:major` for breaking changes.

## Getting Approved Status from Puppet Labs

- Approval requirements are as follows:
  - Solve a unique problem well.
  - Comply with the Puppet Style Guide.
  - Have regular updates from more than one person or organization.
  - Have less than 1 month lag between source repo (GitHub) and Forge.
  - Provide thorough and readable documentation.
  - Be licensed under Apache, MIT, or BSD licenses.
  - Have every standard metadata field filled out, including Puppet version and OS compatibility.
  - Be versioned according to SemVer guidlines.
  - Have rspec and acceptance tests for every manifests.
  - Have unit tests for types, providers, facts, and functions.