In this episode we take a look at how to setup a git deployment workflow.


## Notable Commands

```
git init --bare (this will create a bare git repository with no working directory)
```

## Notable Files

```
# project.git/hooks/post-receive
export GIT_WORK_TREE=/where/your/deployment/files/go
git checkout -f master 
```

## Notable Links

[Git Deployment](http://gitolite.com/the-list-and-irc/deploy.html)