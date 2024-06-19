
|repo命令|等同git命令|备注|
|---|---|---|
|repo init -u|无|初始化|
|repo sync|git pull|同步代码|
|repo upload|git push|上传代码|
|repo forall|无|多仓执行|
|repo start|git checkout -b|创建并切换分支|
|repo checkout|git checkout|切换分支|
|repo status|git status|状态查询|
|repo branches|git branch|分支查询|
|repo diff|git diff|文件对比|
|repo prune|无|删除合并分支|
|repo stage –i|git add --interactive|添加文件到暂存区|
|repo abandon|git branch -D|删除分支|
|repo version|无|查看版本号|


1. 将HEAD强制指向manifest的库，而忽略本地的改动
		`repo sync -d
1. Remove all working directory (and staged) changes
	  `repo forall -c 'git  --hard'
2. Clean untracked files
	  `repo forall -c 'git clean -f -d'
3. 拉代码
	  `repo sync -c`  
