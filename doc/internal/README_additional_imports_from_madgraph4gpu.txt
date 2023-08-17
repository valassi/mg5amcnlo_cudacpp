# Copyright (C) 2020-2023 CERN and UCLouvain.
# Licensed under the GNU Lesser General Public License (version 3 or later).
# Created by: A. Valassi (Aug 2023) for the MG5aMC CUDACPP plugin.

The following describes how the first additional import of changes in the CUDACPP plugin from madgraph4gpu.

The problem is the following (see also https://stackoverflow.com/a/436530):
- In madgraph4gpu there have been several commits A-...-B-...-C (A = first commit, C = current HEAD)
- All changes in epochX/cudacpp/CODEGEN/PLUGIN/CUDACPP_SA_OUTPUT/ between A and B have been filtered out and uploaded to mg5amcnlo_cudacpp
- The additional commits since B and up until C need to be filtered out and uploaded to mg5amcnlo_cudacpp

Commit A is the root commit on master i.e. $(git rev-list --max-parents=0 HEAD), see https://stackoverflow.com/a/1007545:
  dab5d980d

Commit B was the last commit on master when the initial import was performed:
  fe3cdf769

Commit C is presently the last commit on upstream/master
  bd255c01f

1. SQUASH ALL COMMITS BETWEEN A AND B IN THE MADGRAPH4GPU REPO

# Go to my clone of my madgraph4gpu fork on gitlab
cd /data/avalassi/GPU2023/madgraph4gpuMOD

# Checkout B as detached HEAD
# Reset soft to A (root) and amend the commit
lastend=fe3cdf769
git checkout $lastend
git reset --soft $(git rev-list --max-parents=0 HEAD)
git commit --amend -m "Squash initial commits until $lastend"

# Check that there is indeed only one commit left
git log --oneline
  80320501a (HEAD) Squash initial commits until fe3cdf769

# Save the hash for this commit
newstart=$(git log --oneline -n1 | awk '{print $1}')
echo $newstart
  80320501a

2. FILTER OUT ALL COMMITS BETWEEN B AND C ON THE CUDACPP PLUGIN IN THE MADGRAPH4GPU REPO

# Stay in my clone of my madgraph4gpu fork on gitlab
cd /data/avalassi/GPU2023/madgraph4gpuMOD

# Checkout C (upstream/master) as detached HEAD
# Rebase it onto the single commit squashing everything between A and B
git checkout upstream/master
git rebase --onto $newstart $lastend

# Filter out only the plugin files
git filter-branch -f --subdirectory-filter epochX/cudacpp/CODEGEN/PLUGIN/CUDACPP_SA_OUTPUT/

# Check that there are only few commits left (n+1, i.e. n new ones on top of the initial squashed commit)
git log --oneline
  dd5cebb6f (HEAD) [submod] in CODEGEN, configure renaming of CUDACCP_SA_OUTPUT as CUDACPP_OUTPUT when copying to mg5amcnlo
  50d60a02b Squash initial commits until fe3cdf769

# Keep the list of all the additional commits (n=1 here), listed in reverse order
ncommits=$(git log --oneline --reverse | wc -l)
newcommits=$(git log --oneline --reverse -n $((ncommits - 1)) | awk '{print $1}')

3. APPLY (CHERRY-PICK) THESE ADDITIONAL COMMITS ON TOP OF THE MG5AMCNLO_CUDACPP REPO

# Stay in my clone of my madgraph4gpu fork on gitlab
cd /data/avalassi/GPU2023/madgraph4gpuMOD

# Apply these commits on top of the "plugin" branch which includes the past filtered history
# The branch can now be merged onto origin/master: open a MR on github
git checkout plugin
git cherry-pick $newcommits
git push



# Go to my clone of my madgraph4gpu fork on gitlab
cd /data/avalassi/GPU2023/madgraph4gpuMOD
git remote -v
  origin          https://:@gitlab.cern.ch:8443/valassi/madgraph4gpu.git (fetch)
  origin          https://:@gitlab.cern.ch:8443/valassi/madgraph4gpu.git (push)
  upstream        https://github.com/madgraph5/madgraph4gpu.git (fetch)
  upstream        https://github.com/madgraph5/madgraph4gpu.git (push)

# Create and push a new branch plugin in my madgraph4gpu fork from the latest upstream/master
# Note down what was the last commit hash at the time
git fetch master
git checkout -b plugin upstream/master
git push origin -u plugin
git log --oneline -n1 | awk '{print $1}'
  fe3cdf769

# Filter out only the plugin files and move them to the top directory
# Force push to branch plugin in my madgraph4gpu fork
git filter-branch -f --subdirectory-filter epochX/cudacpp/CODEGEN/PLUGIN/CUDACPP_SA_OUTPUT/
git status
  ...
  On branch plugin
  Your branch and 'origin/plugin' have diverged,
  and have 831 and 8041 different commits each, respectively.
git push -f

# Rewrite the commit messages in branch plugin to create links to madgraph4gpu issues
# Force push to branch plugin in my madgraph4gpu fork
git filter-branch -f --msg-filter 'sed -r "s,( |\()#([0-9]),\1madgraph5/madgraph4gpu#\2,g"'
git push -f

3. COMMIT THE MADGRAPH4GPU "PLUGIN" BRANCH TO MY FORK OF THE MG5AMCNLO_CUDACPP REPO

# Go to my clone of the mg5amcnlo_cudacpp repo on github
# This contains a remote of my fork for mg5amcnlo_cudacpp on github
# It also contains a remote of my fork for madgraph4gpu on gitlab
cd /data/avalassi/GPU2023/CUDACPP/mg5amcnlo_cudacpp
git remote -v
  glav    https://:@gitlab.cern.ch:8443/valassi/madgraph4gpu.git (fetch)
  glav    https://:@gitlab.cern.ch:8443/valassi/madgraph4gpu.git (push)
  origin  https://github.com/mg5amcnlo/mg5amcnlo_cudacpp.git (fetch)
  origin  https://github.com/mg5amcnlo/mg5amcnlo_cudacpp.git (push)
  valassi https://github.com/valassi/mg5amcnlo_cudacpp.git (fetch)
  valassi https://github.com/valassi/mg5amcnlo_cudacpp.git (push)

# Checkout (in this mg5amcnlo_cudacpp repo clone) the plugin branch (from the unrelated madgraph4gpu repo)
git checkout -b cudacpp_plugin glav/plugin
git push valassi -u cudacpp_plugin

# Merge the initial mg5amcnlo_cudacpp commit (with README.md) from origin/master onto this branch
# The branch can now be merged onto origin/master: open a MR on github
git merge origin/master --allow-unrelated-histories

(The MR is https://github.com/mg5amcnlo/mg5amcnlo_cudacpp/pull/2 and was eventually merged)
