# ⚡️🤘⚡️Callaway Cloud CI ⚡️🤘⚡️

This guide will outline the steps to setup and use a bitbucket pipeline for auto-deployments.

<img width="1370" alt="callawaycloud___realself___Pipelines_—_Bitbucket" src="https://user-images.githubusercontent.com/5217568/70212789-2d3d1d80-16f5-11ea-8795-93aac82fdde6.png">

_NOTE:_ This setup currently only works with the "Org Development Model" (manifest package.xml).

## 💪 Goals

1. Make it easy to validate changes on Pull Request
2. Handle 75-90% of deployments automatically. Exceptions:
   - Metadata not tracked in source control (email template, named cred, etc)
   - Complex destructive changes
   - Profile configurations
   - Data configuration
3. Only deploy metadata that has changed.
4. Make it impossible to overwrite changes that have been introduced outside of our source control (dang :wombats:)
5. Keep `master` in sync with production. This includes capturing changes to installed "package" versions (via `sfdc-package.json`)
6. Do all of this without overloading the deployment queue

## 💻 Development Workflow

For this CI process to work you just need to follow a one simple rule:

**Master == Production**

Nothing should ever be committed or merged into master unless it has already successfully been deployed. The CI process itself will ensure that this happens.

A typical development flow would look like this:

1. Developer creates feature branch off master
1. Developer builds & commits changes
1. creates PR back into master. The CI will automatically sync production & build the deployment package. This lets us know that there are no merge conflicts as soon as possible.
1. Once the package build successfully the developer can go ahead manually kick off the "Check Package" step. This confirms that the PR is deployable and all tests pass
1. Once the PR has been approved and you are ready to deploy, you can run "Quick Deploy". If the previously checked deployment has been invalidated, you can either rerun the check package step, or manually kick off one of the full pipeline variants.
1. Once the pipeline completes your done! The branch will automatically be merged and cleaned up.

## 🔧 Setup

### Build Scripts

The easiest way to add this pipeline to a project is to use the [callaway yeoman generator](https://github.com/ChuckJonas/generator-ccc):

1. run `npm install -g yo`
2. run `npm install -g generator-ccc`
3. `cd` to project
4. run `yo ccc`

**NOTE:** When running from Windows, the `build/*.sh` files do not seem to retain their executable permission. You'll need to figure out a way to [make the files linux executable](https://unix.stackexchange.com/questions/256964/transferring-executable-files-from-windows-to-linux-via-winscp) for the pipeline ci to run.

The recommended method is to manually grant the files in the [build folder](https://github.com/ChuckJonas/generator-ccc/tree/master/generators/app/templates/static/build) & [bitbucket-pipelines.yml](https://github.com/ChuckJonas/generator-ccc/blob/master/generators/app/templates/static/bitbucket-pipelines.yml) permission to execute.

**Only needed if running from windows** Run the following git command:
`git update-index --chmod=+x (get-item .\build\*.sh).FullName`

### Environment Setup

1. Authorize the production org with sfdx-cli
1. Run `sfdx force:org:display --verbose -u your-prod-user`. Copy the returned "Sfdx Auth Url"
1. Open the repo in bitbucket
1. Navigate to "Settings -> Pipelines -> Settings"
1. Enable Pipelines
1. Navigate to "Settings -> Pipelines -> Repository variables"
1. Create a new variable called AUTH_URL. **MAKE SURE TO CHECK THE SECURE OPTION!!!**

### Scheduled Production Sync

Ideally we want master to always be as close to production as possible. With the CI, every time we deploy, they get synced up. However, if you don't deploy for several days/weeks/months, they will likely get out of sync. We want to not only capture the changes, but also know when the changes happened.

To solve this, we've added a "Scheduled Pipeline" which checks for the last production sync. If it was `> X days`, we will go ahead and run the sync.

**To setup Scheduled Sync:**

1. Navigate to "Pipelines => Schedules" in the repository.
1. click "New Schedule"
1. choose `master` branch
1. choose "Scheduled Production Sync"
1. recommended interval is `daily`
1. select a time for it to run (recommended 3am)
1. click "Create"

<img width="1395" alt="callawaycloud___realself___Pipelines_—_Bitbucket" src="https://user-images.githubusercontent.com/5217568/80502612-1a3e1a80-892e-11ea-9d08-0996cb5bfae8.png">

By default the "sync internal" is set to 3 days. If you want to configure this time to be longer/shorter:

1. Navigate to `Repository Settings` => `Repository Variables`
1. add new variable called `PRODUCTION_SYNC_INTERVAL` with to desired interval (in days)

## 🌊 Pipeline Steps

<img width="300" alt="callawaycloud___ci-example-repo___Pipelines_—_Bitbucket" src="https://user-images.githubusercontent.com/5217568/70215690-4b0d8100-16fb-11ea-95d5-b7f5b0afc2f5.png">

### 1: "Build Package"

**Trigger:** Pull Request Created/Updated

1. It syncs Production into `master` (using [force:source:clean](https://github.com/ChuckJonas/force-source-clean)) and pushes the changes
2. It merges `master` into the current branch
3. It generates an incremental deployment package based on the difference between `master` and the current branch (using [sfdx-git-packager](https://github.com/ChuckJonas/sfdx-git-packager))

After completing, you can inspect the package by downloading the "artifacts".

<img width="800" alt="callawaycloud___ci-example-repo___Pipelines_—_Bitbucket" src="https://user-images.githubusercontent.com/5217568/70214463-b6a21f00-16f8-11ea-9530-87dff421d7b5.png">

### 2: "Check Package"

**Trigger:** Manual

Performs a `--CHECKONLY` deployment with the generated package.

### 3: "Quick Deploy"

**Trigger:** Manual

1. Completes the deployment you previously checked
2. Merges the current branch into `master`

No additional steps are required to close the pull request, although you might want to delete the remote branch.

![CI Resolution Diagram](https://user-images.githubusercontent.com/5217568/93302663-9939cf80-f7b7-11ea-8eb7-2de33dfaec28.png)

## Advanced Control

### Pull-Request Pipeline Options

<img width="959" alt="callawaycloud___realself___Pull_Request__219__Testing_Pipeline_—_Bitbucket" src="https://user-images.githubusercontent.com/5217568/88210040-cbdca700-cc10-11ea-9935-ebdfbdaedbd3.png">

Unfortunately, at the moment it is not possible to pass user variables to a Pull Request Pipeline.

As a work around, we have added the ability to define pipeline variables as text in the PR "description". These "variables start with a `!`, may be place anywhere in the description and are cases insensitive.

**Skip Production Sync**

`!skipSync`

**Run Only Selected Tests**

`!tests=FooTests,BarTests`

### Manual Pipelines

Beyond the pipeline that runs on PR, there are also pipelines that can be run manually:

<img width="539" alt="callawaycloud___realself___Branches_—_Bitbucket" src="https://user-images.githubusercontent.com/5217568/88209549-19a4df80-cc10-11ea-842e-8012a7c6be45.png">
