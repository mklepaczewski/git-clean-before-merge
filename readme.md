   
# What it does

`git pull` and `git merge` won't handle some trivial conflicts:
- you have an untracked file in your working directory and the file exists, and is identical in the merge branch,
- you have a modified file in your working directory and it's identical to the file in the merge branch

Since both the file in the working directory and the one in the merge branch are the same it's safe
to discard local changes and accept the files in the merge branch (since both versions are the same).

This tool cleans working directory of such changes so that `git merge some/branch` can continue.
Cleaning means:
- deleting untracked, identical (to `some/branch`) files from working directory - they'll be added after `git merge some/branch` or `git pull`,
- executing `git checkout the_file` on modified, identical (to `some/branch`) files from working directory - they'll be brought back after `git merge some/branch` or `git pull`    

# Installation

    # Clone it
    git clone https://github.com/mklepaczewski/git-clean-before-merge.git
    
    # Make it executable
    cd git-clean-before-merge
    chmod +x git-clean-before-merge
    
    # Add a symbolic link from /usr/local/bin/git-clean-before-merge
    ln -s "$PWD/git-clean-before-merge" /usr/local/bin/git-clean-before-merge

# Cmd line parameters

    git-clean-before-merge [--pretend] [--debug] [branch/to/be/merged-or-pulled]
    
 - `--pretend` - write what the program is going to do, without actually doing anything.
   No files will be modified.
 - `--debug` - print debug messages
 - `branch/to/be/merged-or/pulled` - the branch to run the cleanup against  

# Output format

Taken from the Example section:

    # git-clean-before-merge --pretend dev
    [C] Identical with remote, checking out a.txt
    [R] Untracked file found and identical with remote tip, removing: b.txt
    
- `[C]` - `git checkout the_file` will be run,
- `[R]` - the file will be deleted
  
# Example

Let's create an `example` repo and add some files to it (`a.txt`):

    # cd ~
    # mkdir example
    # cd example
    # git init .
    Initialized empty Git repository in /home/matt/example/.git/
    # echo "Hello World" > a.txt
    # git add a.txt
    # git commit -m "Initial commit"
    [master (root-commit) 2ebf5cc] Initial commit
     1 file changed, 1 insertion(+)
     create mode 100644 a.txt

Now, let's switch branch to `dev` and add `b.txt` there, and modify `a.txt`:

    # git checkout -b dev
    Switched to a new branch 'dev'
    # echo "Hello, World" > a.txt
    # echo "I'm alive" > b.txt
    # git checkout master^C
    # git status
    On branch dev
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git checkout -- <file>..." to discard changes in working directory)
    
            modified:   a.txt
    
    Untracked files:
      (use "git add <file>..." to include in what will be committed)
    
            b.txt
    
    no changes added to commit (use "git add" and/or "git commit -a")
    
    # git add .
    # git commit -m "Changes"
    [dev 900a7ad] Changes
     2 files changed, 2 insertions(+), 1 deletion(-)
     create mode 100644 b.txt

Switch back to `master` and try to merge `dev`:

    # git checkout master
    Switched to branch 'master'
    # echo "Hello, World" > a.txt
    # echo "I'm alive" > b.txt
    # git merge dev
    Updating 2ebf5cc..900a7ad
    error: Your local changes to the following files would be overwritten by merge:
            a.txt
    Please commit your changes or stash them before you merge.
    error: The following untracked working tree files would be overwritten by merge:
            b.txt
    Please move or remove them before you merge.
    Aborting

Ooops, we have a conflict, even though our local `a.txt` and `b.txt` are identical to `dev/a.txt` and `dev/b.txt`.  Let's
run `git-clean-before-merge --pretend` to see what we can do about it:

    # git-clean-before-merge --pretend dev
    [C] Identical with remote, checking out a.txt
    [R] Untracked file found and identical with remote tip, removing: b.txt

It'll revert `a.txt` and remove `b.txt` so that `git merge dev` can continue.
    
    # git-clean-before-merge dev
    [C] Identical with remote, checking out a.txt
    [R] Untracked file found and identical with remote tip, removing: b.txt
    
    # git status
    On branch master
    nothing to commit, working tree clean
    
    # git merge dev
    Updating 2ebf5cc..900a7ad
    Fast-forward
     a.txt | 2 +-
     b.txt | 1 +
     2 files changed, 2 insertions(+), 1 deletion(-)
     create mode 100644 b.txt

# Todo list
 - add examples ;-)
 - error and sanity checks,
 - don't rely on bash
 - make sure that all commands that are going to be used do exist (git)
 - instead of checking if modified/untracked file is the same as on the tip of the merge branch
   check if it exists in any of commits between our local commit and the tip of the merge branch.
   If it exists in any of those commits then most probably we can discard our local changes, as
   at some point in time in the repo history we had our current state and we updated on it, so it
   should be safe to use the most recent version of the file
   Also, if it has been removed then:
   - revert it to original state (git checkout the_file) if it's a tracked file
   - remove the file if it's an untracked file
 - rename debug_msg() to msg_debug()

# Is it safe to use it on production?

- No. First, make yourself familiar with the tool in dev environment.
- The tool may revert / delete some files from your working directory before you execute `git merge`. This may
  mess up your environement. If you know it's ok, then it's ok. If you're unsure, then it's not ok. 
- you shouldn't be having the kind of issues this tool solves in production environment. You might be doing something wrong. 

