# git-mirror
this is my git mirror server build style. I set the hooks script only the slave repository.

now configure file only ...

#### environment
```
                          -> 'fetch'
 [github]+------------+[mirror](clone --mirror)
   | (no hooks)         | <- 'push'
   |                    |
   |                    | hooks/                +   crontab
   |                    |   pre-receive  *[1]         fetch
   |                    |   post-receive *[2]
   |                    |
   |                    |
 [clone-01]           [clone-02]
 (direct)             (direct from mirror)
```
mirror test repo : git@github.com:kotaro-dev/timechk.git

#### configuration

1.make mirror server
 (login mirror server)
``` 
 mkdir gitmrr && cd gitmrr
 git clone --mirror git@github.com:kotaro-dev/timechk.git
 (you can see timechk.git directory. ls -alF)
```
2.setting hooks script
 (in mirror server)
```
 cd timechk.git
 (ls -alF hooks/)
 cp hooks/post-update.sample hooks/pre-receive
 cp hooks/post-update.sample hooks/post-receive
 
 vim hooks/pre-receive
 ----
 #!/bin/sh
master_head=`git ls-remote --heads -q|sed -n -e /master/p|cut -f 1`
slave_head=`git show-ref --heads|sed -n -e /master/p|cut -f 1 -d " "`
if [ "${master_head}" != "${slave_head}" ]; then
  nohup git fetch --all > /dev/null < /dev/null &
  exit -1
fi
 ----
 
 vim hooks/post-receive
 ----
 #!/bin/sh
 nohup git push --mirror & >/dev/null </dev/null &
----
```
3.configure .ssh/config
 (in mirror server)
```
 vim ~/.ssh/config
 ----
 ControlMaster auto
 ControlPath /tmp/%r@%h:%p
 ControlPersist yes
 ----
```

#### simple test operation

```
 (1)[github]+--+[mirror](2) gstcore121 ~/gitmrr/timechk.git
      |           |
 (3)[clone1]    [clone2](4)
```
(preparation)
```
 (1): create new repository for test - timechk
 (2): git clone --mirror git@github.com:kotaro-dev/timechk.git
 (3): git clone git@github.com:kotaro-dev/timechk.git
 (4): git clone core@gstcore121:~/gitmrr/timechk.git
```
(test operation)
```
 (3): echo "mst file01">mst01.txt  && git add mst01.txt && git commit -m "master file01"
 (3): git push 

 (4): echo "mrr file01">mrr01.txt  && git add mrr01.txt && git commit -m "mirror file01"
 (4): git push 
 (you will see error message like you need pull first. - from pre-receive)

 (4): git pull
 (4): git push
 (you will get success push into mirror server.)
 
 (3): git pull
 (you will get mrr01.txt from github(master server).)
```
