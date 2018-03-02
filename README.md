
# Use git instead of subversion for puppet dynamic environments

## Description
Support puppet branch development using git as replacement for subversion. Because of the natural of how subversion works using separate directories for branching, current branching workflow does not work because git branches are 'in place' using current directory structure.

This script will detect puppet branches from origin and create git sparse checkouts of the branches, then links them to the environments.
This runs on the puppet server.

## Requirement
ssh keys already setup to all access to your git puppet repository.

your git puppet repository directory structure is
* /environments
* /modules

You are git cloning to /etc/puppetlabs/code
e.g. git clone [your_git_repository] /etc/puppetlabs/code

## Overview
* scans git origin for all branches
* git sparse check out to ${puppet_branch_path} directory
* symlinks ${puppet_master_path}/environment/[branch] to  ${puppet_branch_path}/[branch]/environments/production

## Instructions
* cut and past this script to your host. filename: /usr/local/bin/create_env.sh
```
#!/bin/bash

puppet_branch_path='/opt/puppet_branch'                     # git sparse check out branch
puppet_master_path='/etc/puppetlabs/code'                   # git clone master
git_repo='[your_git_repository]'


cleanup_branch() {

  for i in ${current_envs}
  do
    echo $i
    valid_dir=false

    for j in ${envs}
    do
      echo $j
      if [ $i == $j ]
      then
        valid_dir=true
        break
      fi
    done
    if [ ${valid_dir} == 'false' ]
    then
      rm -rf ${puppet_branch_path}/${i}
    fi
  done
}
cleanup_link() {

  for i in ${current_links}
  do
    echo $i
    valid_dir=false

    for j in ${envs}
    do
      echo $j
      if [ $i == $j ]
      then
        valid_dir=true
        break
      fi
    done
    if [ ${valid_dir} == 'false' ]
    then
      unlink ${puppet_master_path}/environments/${i}
    fi
  done
}

mkdir -p ${puppet_branch_path}

current_envs=`ls -l ${puppet_branch_path} | grep '^d' | awk '{print $9}'`
current_links=`ls -l ${puppet_master_path}/environments| grep '^l' | awk '{print $9}'`
cd "${puppet_master_path}"
git remote prune origin
envs=`git branch -r | grep '  origin' | egrep -v 'HEAD|git-svn|master' | awk '{split($1,a,"/"); print a[2]}'`

cleanup_branch
cleanup_link

for i in ${envs}
do
  branch_path="${puppet_branch_path}/${i}"
  if [ -d "${puppet_branch_path}/${i}" ]
  then
    cd ${puppet_branch_path}/${i}
    git pull
  else
    mkdir -p ${branch_path}
    cd ${branch_path}
    git init
    git config core.sparsecheckout true
    git remote add -f origin ${git_repo}
    echo "environments/production" > .git/info/sparse-checkout
    git checkout ${i}
    cd ${puppet_master_path}/environments
    unlink ${i}
    ln -s ${puppet_branch_path}/${i}/environments/production ${i}
  fi
done

```

* run script
```
sh -x /usr/local/bin/create_env.sh
```
* Periodically run this script via crontab , puppet agent -t, or git hook

## Workflow
### Overview
* git clone on your local machine
* Create a branch and edit your code
* Check in and push branch
* Run puppet agent on target server
* Merge branch to master and push to origin
* Clean up the dynamic environment by removing the git branch

### git clone on your local machine
``
git clone [url_to_your_puppet_git_repo]
```

### Create a branch and edit your code
Only work on **environments/production**
```
git pull
git branch [your_branch_name]
git checkout [your_branch_name]
cd environments/production
# Edit your code
```
Note: DO NOT add these directories 'Untracked files'. only environments/production**
```
# On branch master
# Untracked files:
# (use` `"git add <file>..."`  `to include in what will be committed)
#
# environments/test1
# environments/test2
# environments/xxxxx
```
### Check in and push branch
```
git status
git add [file / directories] # DO NOT ADD environments/xxxx
git status
git commit --author="Name <email>" -m "[ticket] comment"
git push origin [your_branch_name]
```
### Run puppet agent on target server
Depending on how you implemented create_env.sh. This needs to be run to update the puppet master.
Once the puppet master has the update, then run the puppet client.
```
ssh root@[target_server]
puppet agent -t
```
###   Merge branch to master and push to origin
When you are completed satisfy with the work on the branch and testing is finalized, merge the branch to master
```
git checkout master
git pull -a
git checkout [your_branch_name]
git merge master
git checkout master
git merge --squash [your_branch_name]
git push origin master
```
### Clean up the dynamic environment by removing the git branch
```
git branch -D [your_branch_name]
git push origin :[your_branch_name]
git remote prune origin
```
Note: Next run of create_env.sh will remove the branch from the puppet server.
