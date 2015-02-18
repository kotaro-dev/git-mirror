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
   |                    |   pre-receive   *[1]         fetch
   |                    |   post-receive  *[2]
   |                    |   git-fetch-all *[3]
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

   # if os use systemctl, cant work nohup xxx &. so ...
   chksysd=`systemctl --version` > /dev/null 2>&1
   if [ "${chksysd}" != "" ]; then
     nohup sudo systemd-run ${fpath}/git-fetch-all > /dev/null 2>&1 < /dev/null &
   else
     nohup git fetch --all > /dev/null 2>&1 < /dev/null &
   fi

   exit -1
 fi
 ----
 
 vim hooks/post-receive
 ----
 #!/bin/sh
 git push --mirror > /dev/null 2>&1 < /dev/null
 ----
 
 (if work on systmctl)
 vim hooks/git-fetch-all
 ----
 #!/bin/sh
 mypath=${0%/*}
 uppath=${mypath%/*}
 cd ${uppath}
 sudo -u core -H git fetch --all > /dev/null 2>&1 < /dev/null
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

#### modification

Change for systemctl system.  
in this os, i can't use 'nohup xxx &' for working in the background.  
So i use systemd-run to work a deamon.  

#### note

 coreos に対して ssh 接続して background job を実行して exit すると  
 background 処理のはずが、kill されて想定通りの動作が行われなかった。  
 最終的に、systemd-run で 処理を実行することで 希望通りの 動作を実現できた。  
 ×: nohup git fetch --all &  
 ○: nohup sudo systemd-run git fetch --all &  
    (need change directory in your git bare repo.)   
 参考までに、systemd-run で実行して 正常終了以外の status で終わると failed log が残り  
 teraterm等で接続するたびに failed したservice のリストが表示される。  
 sudo systemctl reset-failed  
 上記コマンドで reset 出来る。(反映させるのに少々時間がかかった。)  
 
