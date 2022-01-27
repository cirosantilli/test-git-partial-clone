# test-git-partial-clone
Minimal Git repository to illustrate partial clones (--filter=blob:none): 
https://stackoverflow.com/questions/600079/how-do-i-clone-a-subdirectory-only-of-a-git-repository/52269934#52269934 
This repo is programatically and deterministically generated as explained there.

# How do I clone a subdirectory only of a Git repository?

**`git clone --filter` from git 2.19 now works on GitHub (tested 2021-01-14, git 2.30.0)**

This option was added together with an update to the remote protocol, and it truly prevents objects from being downloaded from the server.

E.g., to clone only objects required for `d1` of this minimal test repository: https://github.com/cirosantilli/test-git-partial-clone I can do:

```
git clone \
  --depth 1  \
  --filter=blob:none  \
  --sparse \
  https://github.com/cirosantilli/test-git-partial-clone \
;
cd test-git-partial-clone
git sparse-checkout set d1
```

Here's a less minimal and more realistic version at https://github.com/cirosantilli/test-git-partial-clone-big-small

```
git clone \
  --depth 1  \
  --filter=blob:none  \
  --sparse \
  https://github.com/cirosantilli/test-git-partial-clone-big-small \
;
cd test-git-partial-clone-big-small
git sparse-checkout set small
```

That repository contains:

- a big directory with 10 10MB files
- a small directory with 1000 files of size one byte

All contents are pseudo-random and therefore incompressible.

Clone times on my 36.4 Mbps internet:

- full: 24s
- partial: "instantaneous"

The `sparse-checkout` part is also needed unfortunately. You can also only download certain files with the much more understandable:

```  
git clone \
  --depth 1  \
  --filter=blob:none  \
  --no-checkout \
  https://github.com/cirosantilli/test-git-partial-clone \
;
cd test-git-partial-clone
git checkout master -- d1
```

but that method for some reason [downloads files one by one very slowly](https://github.com/isaacs/github/issues/1888), making it unusable unless you have very few files in the directory.

**Analysis of the objects in the minimal repository**

The clone command obtains only:

- a single [commit object](https://stackoverflow.com/questions/22968856/what-is-the-file-format-of-a-git-commit-object-data-structure/37438460#37438460) with the tip of the `master` branch
- all 4 [tree objects](https://stackoverflow.com/questions/14790681/what-is-the-internal-format-of-a-git-tree-object) of the repository:
  - toplevel directory of commit
  - the the three directories `d1`, `d2`, `master`

Then, the `git sparse-checkout set` command fetches only the missing blobs (files) from the server:

- `d1/a`
- `d1/b`

Even better, later on GitHub will likely start supporting:

```
  --filter=blob:none \
  --filter=tree:0 \
```

where [`--filter=tree:0` from Git 2.20](https://github.com/git/git/blob/master/Documentation/RelNotes/2.20.0.txt#L132) will prevent the unnecessary `clone` fetch of all tree objects, and allow it to be deferred to `checkout`. But on my 2020-09-18 test that fails with:

```
fatal: invalid filter-spec 'combine:blob:none+tree:0'
```

presumably because the `--filter=combine:` composite filter (added in Git 2.24, implied by multiple `--filter`) is not yet implemented.

I observed which objects were fetched with:

```
git verify-pack -v .git/objects/pack/*.pack
```

as mentioned at: https://stackoverflow.com/questions/7348698/git-how-to-list-all-objects-in-the-database/18793029#18793029 It does not give me a super clear indication of what each object is exactly, but it does say the type of each object (`commit`, `tree`, `blob`), and since there are so few objects in that minimal repo, I can unambiguously deduce what each object is.

`git rev-list --objects --all` did produce clearer output with paths for tree/blobs, but it unfortunately fetches some objects when I run it, which makes it hard to determine what was fetched when, let me know if anyone has a better command. 

TODO find GitHub announcement that saying when they started supporting it. https://github.blog/2020-01-17-bring-your-monorepo-down-to-size-with-sparse-checkout/ from 2020-01-17 already mentions `--filter blob:none`.

**`git sparse-checkout`**

I think this command is meant to manage a settings file that says "I only care about these subtrees" so that future commands will only affect those subtrees. But it is a bit hard to be sure because the current documentation is a bit... sparse ;-)

It does not, by itself, prevent the fetching of blobs.

If this understanding is correct, then this would be a good complement to `git clone --filter` described above, as it would prevent unintentional fetching of more objects if you intend to do git operations in the partial cloned repo.

When I tried on Git 2.25.1:

```
git clone \
  --depth 1 \
  --filter=blob:none \
  --no-checkout \
  https://github.com/cirosantilli/test-git-partial-clone \
;
cd test-git-partial-clone
git sparse-checkout init
```

it didn't work because the `init` actually fetched all objects.

However, in Git 2.28 it didn't fetch the objects as desired. But then if I do:

```
git sparse-checkout set d1
```

`d1` is not fetched and checked out, even though this explicitly says it should: https://github.blog/2020-01-17-bring-your-monorepo-down-to-size-with-sparse-checkout/#sparse-checkout-and-partial-clones With disclaimer:

> Keep an eye out for the partial clone feature to become generally available[1].
>
> [1]: GitHub is still evaluating this feature internally while it’s enabled on a select few repositories (including the example used in this post). As the feature stabilizes and matures, we’ll keep you updated with its progress.

So yeah, it's just too hard to be certain at the moment, thanks in part to the joys of GitHub being closed source. But let's keep an eye on it.

**Command breakdown**

The server should be configured with:

    git config --local uploadpack.allowfilter 1
    git config --local uploadpack.allowanysha1inwant 1

Command breakdown:

- `--filter=blob:none` skips all blobs, but still fetches all [tree objects](https://stackoverflow.com/questions/14790681/what-is-the-internal-format-of-a-git-tree-object/37105125#37105125)
- `--filter=tree:0` skips the unneeded trees: https://www.spinics.net/lists/git/msg342006.html
- `--depth 1` already implies `--single-branch`, see also: https://stackoverflow.com/questions/1778088/how-to-clone-a-single-branch-in-git
- `file://$(path)` is required to overcome `git clone` protocol shenanigans: https://stackoverflow.com/questions/47307578/how-to-shallow-clone-a-local-git-repository-with-a-relative-path
- `--filter=combine:FILTER1+FILTER2` is the syntax to use multiple filters at once, trying to pass `--filter` for some reason fails with: "multiple filter-specs cannot be combined". This was added in Git 2.24 at e987df5fe62b8b29be4cdcdeb3704681ada2b29e "list-objects-filter: implement composite filters"

  Edit: on Git 2.28, I experimentally see that `--filter=FILTER1 --filter FILTER2` also has the same effect, since GitHub does not implement `combine:` yet as of 2020-09-18 and complains `fatal: invalid filter-spec 'combine:blob:none+tree:0'`. TODO introduced in which version?

The format of `--filter` is documented on `man git-rev-list`.

Docs on Git tree:

- https://github.com/git/git/blob/v2.19.0/Documentation/technical/partial-clone.txt
- https://github.com/git/git/blob/v2.19.0/Documentation/rev-list-options.txt#L720
- https://github.com/git/git/blob/v2.19.0/t/t5616-partial-clone.sh

**Test it out locally**

The following script reproducibly generates the https://github.com/cirosantilli/test-git-partial-clone repository locally, does a local clone, and observes what was cloned:

```
#!/usr/bin/env bash
set -eu

list-objects() (
  git rev-list --all --objects
  echo "master commit SHA: $(git log -1 --format="%H")"
  echo "mybranch commit SHA: $(git log -1 --format="%H")"
  git ls-tree master
  git ls-tree mybranch | grep mybranch
  git ls-tree master~ | grep root
)

# Reproducibility.
export GIT_COMMITTER_NAME='a'
export GIT_COMMITTER_EMAIL='a'
export GIT_AUTHOR_NAME='a'
export GIT_AUTHOR_EMAIL='a'
export GIT_COMMITTER_DATE='2000-01-01T00:00:00+0000'
export GIT_AUTHOR_DATE='2000-01-01T00:00:00+0000'

rm -rf server_repo local_repo
mkdir server_repo
cd server_repo

# Create repo.
git init --quiet
git config --local uploadpack.allowfilter 1
git config --local uploadpack.allowanysha1inwant 1

# First commit.
# Directories present in all branches.
mkdir d1 d2
printf 'd1/a' > ./d1/a
printf 'd1/b' > ./d1/b
printf 'd2/a' > ./d2/a
printf 'd2/b' > ./d2/b
# Present only in root.
mkdir 'root'
printf 'root' > ./root/root
git add .
git commit -m 'root' --quiet

# Second commit only on master.
git rm --quiet -r ./root
mkdir 'master'
printf 'master' > ./master/master
git add .
git commit -m 'master commit' --quiet

# Second commit only on mybranch.
git checkout -b mybranch --quiet master~
git rm --quiet -r ./root
mkdir 'mybranch'
printf 'mybranch' > ./mybranch/mybranch
git add .
git commit -m 'mybranch commit' --quiet

echo "# List and identify all objects"
list-objects
echo

# Restore master.
git checkout --quiet master
cd ..

# Clone. Don't checkout for now, only .git/ dir.
git clone --depth 1 --quiet --no-checkout --filter=blob:none "file://$(pwd)/server_repo" local_repo
cd local_repo

# List missing objects from master.
echo "# Missing objects after --no-checkout"
git rev-list --all --quiet --objects --missing=print
echo

echo "# Git checkout fails without internet"
mv ../server_repo ../server_repo.off
! git checkout master
echo

echo "# Git checkout fetches the missing directory from internet"
mv ../server_repo.off ../server_repo
git checkout master -- d1/
echo

echo "# Missing objects after checking out d1"
git rev-list --all --quiet --objects --missing=print
```

[GitHub upstream](https://github.com/cirosantilli/test-git-web-interface/blob/296dc8b8edc051299bfa5404bb84f79e69ea1cde/other-test-repos/partial-clone.sh).

Output in Git v2.19.0:

```
# List and identify all objects
c6fcdfaf2b1462f809aecdad83a186eeec00f9c1
fc5e97944480982cfc180a6d6634699921ee63ec
7251a83be9a03161acde7b71a8fda9be19f47128
62d67bce3c672fe2b9065f372726a11e57bade7e
b64bf435a3e54c5208a1b70b7bcb0fc627463a75 d1
308150e8fddde043f3dbbb8573abb6af1df96e63 d1/a
f70a17f51b7b30fec48a32e4f19ac15e261fd1a4 d1/b
84de03c312dc741d0f2a66df7b2f168d823e122a d2
0975df9b39e23c15f63db194df7f45c76528bccb d2/a
41484c13520fcbb6e7243a26fdb1fc9405c08520 d2/b
7d5230379e4652f1b1da7ed1e78e0b8253e03ba3 master
8b25206ff90e9432f6f1a8600f87a7bd695a24af master/master
ef29f15c9a7c5417944cc09711b6a9ee51b01d89
19f7a4ca4a038aff89d803f017f76d2b66063043 mybranch
1b671b190e293aa091239b8b5e8c149411d00523 mybranch/mybranch
c3760bb1a0ece87cdbaf9a563c77a45e30a4e30e
a0234da53ec608b54813b4271fbf00ba5318b99f root
93ca1422a8da0a9effc465eccbcb17e23015542d root/root
master commit SHA: fc5e97944480982cfc180a6d6634699921ee63ec
mybranch commit SHA: fc5e97944480982cfc180a6d6634699921ee63ec
040000 tree b64bf435a3e54c5208a1b70b7bcb0fc627463a75    d1
040000 tree 84de03c312dc741d0f2a66df7b2f168d823e122a    d2
040000 tree 7d5230379e4652f1b1da7ed1e78e0b8253e03ba3    master
040000 tree 19f7a4ca4a038aff89d803f017f76d2b66063043    mybranch
040000 tree a0234da53ec608b54813b4271fbf00ba5318b99f    root

# Missing objects after --no-checkout
?f70a17f51b7b30fec48a32e4f19ac15e261fd1a4
?8b25206ff90e9432f6f1a8600f87a7bd695a24af
?41484c13520fcbb6e7243a26fdb1fc9405c08520
?0975df9b39e23c15f63db194df7f45c76528bccb
?308150e8fddde043f3dbbb8573abb6af1df96e63

# Git checkout fails without internet
fatal: '/home/ciro/bak/git/test-git-web-interface/other-test-repos/partial-clone.tmp/server_repo' does not appear to be a git repository
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

# Git checkout fetches the missing directory from internet
remote: Enumerating objects: 1, done.
remote: Counting objects: 100% (1/1), done.
remote: Total 1 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (1/1), 45 bytes | 45.00 KiB/s, done.
remote: Enumerating objects: 1, done.
remote: Counting objects: 100% (1/1), done.
remote: Total 1 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (1/1), 45 bytes | 45.00 KiB/s, done.

# Missing objects after checking out d1
?8b25206ff90e9432f6f1a8600f87a7bd695a24af
?41484c13520fcbb6e7243a26fdb1fc9405c08520
?0975df9b39e23c15f63db194df7f45c76528bccb
```

Conclusions: all blobs from outside of `d1/` are missing. E.g. `0975df9b39e23c15f63db194df7f45c76528bccb`, which is `d2/b` is not there after checking out `d1/a`.

Note that `root/root` and `mybranch/mybranch` are also missing, but `--depth 1` hides that from the list of missing files. If you remove `--depth 1`, then they show on the list of missing files.

**I have a dream**

This feature could revolutionize Git.

Imagine having all the code base of your enterprise [in a single repo](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/fulltext) without [ugly third-party tools like `repo`](https://stackoverflow.com/questions/1809774/how-to-compile-the-android-aosp-kernel-and-test-it-with-the-android-emulator/48310014#48310014).

Imagine [storing huge blobs directly in the repo without any ugly third party extensions](https://stackoverflow.com/questions/540535/managing-large-binary-files-with-git/53652983#53652983).

Imagine if GitHub would allow [per file / directory metadata](https://github.com/isaacs/github/issues/49) like stars and permissions, so you can store all your personal stuff under a single repo.

Imagine if [submodules were treated exactly like regular directories](https://stackoverflow.com/questions/34037219/why-were-gnu-binutils-and-gdb-merged-as-one-package): just request a tree SHA, and a [DNS-like mechanism resolves your request](https://github.com/ipfs/ipfs), first looking on your [local `~/.git`](https://stackoverflow.com/questions/2350996/how-can-one-safely-use-a-shared-object-database-in-git), then first to closer servers (your enterprise's mirror / cache) and ending up on GitHub.
