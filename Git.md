# Work Tree

- To create a local worktree tracking a remote branch, use:
```
git worktree add -b <LOCAL BRANCH> <LOCAL PATH> <REMOTE BRANCH>
```

# LFS

[HERE](https://github.com/git-lfs/git-lfs/blob/master/docs/man/git-lfs-config.5.ronn) is a list of all the config settings for git-lfs.

Some common settings you may want to change locally to improve performance:
```
git config --unset lfs.fetchrecentalways
git config --add   lfs.fetchrecentalways true

git config --unset lfs.fetchrecentrefsdays
git config --add   lfs.fetchrecentrefsdays <NUMBER|default is 7>

git config --unset lfs.concurrenttransfers
git config --add   lfs.concurrenttransfers <NUMBER|default is 8>
```
