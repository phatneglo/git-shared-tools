# Duplication Process (.git)

1. First, let's save your current state as "san-fernando":

   ```
   git checkout -b san-fernando
   git add .
   git commit -m "Save current state as san-fernando"
   git push -u origin san-fernando
   ```

2. Now, create and switch to the "leyte" branch:

   ```
   git checkout -b leyte
   ```

3. Make your desired changes for the "leyte" version.

4. After making changes, commit them:

   ```
   git add .
   git commit -m "Create leyte version"
   git push -u origin leyte
   ```

5. To switch between versions:

   - For san-fernando: `git checkout san-fernando`
   - For leyte: `git checkout leyte`

This setup allows you to easily switch between the two versions. When you checkout a branch, your working directory will reflect the state of that branch.

If you need to make further changes to either version, just checkout the appropriate branch, make your changes, and commit them.

To see which branch you're currently on:

```
git branch
```

Remember, if you have uncommitted changes when trying to switch branches, Git will either try to merge them or prevent you from switching. It's a good practice to commit or stash your changes before switching branches.

## Dev Server Duplicate (Only when creating Snapshot!)

To create a new DigitalOcean droplet with similar settings to an existing one, but with different database content, you can follow these steps:

1. Create a snapshot of your existing droplet:
   - Log into your DigitalOcean account
   - Go to the Droplets section
   - Select the droplet you want to replicate
   - Click on "Snapshots" in the left sidebar
   - Create a new snapshot (this may take a few minutes)

2. Create a new droplet from the snapshot:
   - Go to the "Create" menu and select "Droplets"
   - In the "Choose an image" section, select the "Snapshots" tab
   - Choose the snapshot you just created
   - Select the desired size, region, and additional options
   - Create the new droplet

3. Once the new droplet is created, access it via SSH

4. Update the database configuration:
   - Modify the database connection settings in your application to point to a new, empty database
   - Create a new database or modify the existing one as needed

5. Clear or modify the existing database content:
   - If you want to start with an empty database, you can drop and recreate it
   - If you need to modify the content, you can run SQL scripts to update, delete, or insert new data

6. Test your application to ensure it's working with the new database setup

This approach allows you to keep all the configurations and installed software from your original droplet while giving you a fresh start with the database content.

## Git Setup

To download the "leyte" branch onto your dev server where you currently have a snapshot of the main branch, you can follow these steps:

1. First, connect to your dev server via SSH if you haven't already.

2. Navigate to your project directory:

   ```
   cd /path/to/your/project
   ```

3. Ensure you're on the main branch and your working directory is clean:

   ```
   git checkout main
   git status
   ```

   If there are any uncommitted changes, either commit them or stash them.

4. Fetch the latest changes from the remote repository:

   ```
   git fetch origin
   ```

   This command updates your local repository with information about all branches on the remote, including "leyte".

5. Create a local "leyte" branch that tracks the remote "leyte" branch:

   ```
   git checkout -b leyte origin/leyte
   ```

   This command creates a new local branch named "leyte" and sets it up to track the remote "leyte" branch. It also switches you to this new branch.

6. Pull the latest changes from the remote "leyte" branch:

   ```
   git pull origin leyte
   ```

   This ensures your local "leyte" branch is up-to-date with the remote version.

Now you have the "leyte" branch on your dev server, and you're currently on that branch. You can switch between "main" and "leyte" using:

```
git checkout main   # to switch to main
git checkout leyte  # to switch to leyte
```

Remember, if you make any changes to the "leyte" branch on your dev server and want to push them back to the remote repository, you can use:

```
git push origin leyte
```

Is there anything else you'd like me to clarify about this process?
