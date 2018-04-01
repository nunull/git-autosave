# git-autosave

An experiment in automatically backing up the working directory in a git
repository whenever a file changes.

## Limitations

This is an experiment and might break. `git-autosave` should work on Linux
and macOS. Automatically initializing it is only available for Bash and ZSH.

## Installation

1. Clone this and link or move `./git-autosave` into your `$PATH`.
2. Optionally add `eval "$(git autosave init)"` to your `.bashrc`,
   `.bash_profile` or `.zshrc`. If you skip this step, `git-autosave` will not
   be automatically initialized. You can initialize it by running `eval "$(git
   autosave init)"` in a shell session or use `git-autosave` manually by
   running `git autosave save-working-dir` each time you want to create a
   backup.

## Usage

After installing, use automatic mode to automatically create backups or manual
mode and call `git autosave save-working-dir` to manually create backups.

Use `git autosave list` to list all backups. To restore the working directory
to a backups state, copy the hash of the backup from `git autosave list` and 
use `git autosave restore <hash>`. This will restore the working directory to
the backups state.

### Automatic vs Manual Mode

In automatic mode, `git-autosave` checks for changes each time a shell prompt
is displayed and creates a new backup commit if necessary. In manual mode, the
user has to call `git autosave save-working-dir` manually.

See the *installation* section on how to set up automatic mode.

## How it Works

### Creating Backups

Backups are created either by calling `git autosave save-working-dir` manually
or by using automatic mode, which checks for changes each time a shell prompt
is displayed and calls `save-working-dir` if it detects changes. See the
section on *Automatic vs Manual Mode* below.

A backup is simply a commit with "the contents of the working directory". The
latest backup commit is stored in `refs/autosave`.

The process to create a backup is as follows:

1. First, a tree object representing the current working directory is created.
   This step is needed to be able to create a commit *without touching the
   index*. The index needs to be kept as is, since otherwise `git-autosave`
   would break normal git workflows.<br>
   To create a tree object from the working directory, a function iterates over
   every file in the working directory and writes every file as a blob object
   using `git hash-object`.  The resulting hashes are accumulated in a `git
   ls-tree` output formatted string. For each directory, a tree entry is added
   by calling the iterator function recursively. The resulting tree is written
   using `git mktree`.
2. Secondly, a commit from the previously created tree object is created. `git
   commit` - which is normally used to create a commit - uses the index to
   figure out what to commit. Since, we don't want to touch the index, `git
   commit-tree`, is used. This directly commits a tree object.<br>
   The parent of the commit is either left out if the commit is the first
   backup commit, or set to whatever `refs/autosave` points to.
3. Lastly, `refs/autosave` is updated to point to the newly created backup
   commit.

See also `git help hash-object`, `git help mktree`, `git help show-ref`, `git
help commit-tree` and `git help update-ref`.

### Restoring Backups

The working directoy can be restored to a previous backup state by referencing
a backup commit by it's hash. Other ways of specifying revisions as found in
the *specifying revisions* section of the `gitrevisions` documentation can't be
used. If other ways of specifying a revision like the `<refname>` would be
allowed, it would not be nicely controllable by the user what revision the
working directory is reset to, since `git-autosave` might have created a new
backup commit and changed `refs/autosave` since the user viewed the backup
state.

The process to restore a backup is as follows:

1. First, a patch is created by:
    1. Creating a tree object from the working directory. This uses the same
       process as described in the section about _creating backups_.
    2. Creating a commit from that tree object.
    3. Diffing the created tree and the tree of the commit to be restored. This
       is done by `git diff-tree`.
2. The resulting patch is applied to the working directory.

See also `git help apply`, `git help diff-tree` and `git help revisions`.

## Environment Variables

- `GIT_AUTOSAVE_DEBUG`: If this is set, `git-autosave` prints more information
  useful for debugging or understand what's going on. This can be escpecially
  useful in automatic mode.
