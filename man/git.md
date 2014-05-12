deployment git manual
======

Deployment GIT

Installation
------------
https://github.com/oscm/deployment.git

	$ git clone https://github.com/oscm/deployment.git
	$ chmod 755 -R deployment
	$ export DEPLOY_HOME=~/deployment
  	
临时使用的方法

	export DEPLOY_HOME=/home/user/deployment
		
	$ cd deployment/
	$ ln -s bin/deploy.git run
		
使用说明

	$ ./run
	Usage: ./run [OPTION] <server-id> <directory/timepoint>
	
	OPTION:
		development <domain> <host>
		testing <domain> <host>
		production <domain> <host>
	
		branch {development|testing|production} <domain> <host> <branchname>
		revert {development|testing|production} <domain> <host> <revision>
		backup <domain> <host> <directory>
		release <domain> <host> <tags> <message>
	
		list
		list <domain> <host>
	
		clean {development|testing|production} <domain> <host>
		log <project> <line>
	
		conf list
		cron show
		cron setup
		cron edit
		
		
模拟演示

环境说明

development 开发环境

testing 测试环境，代码来自开发环境的合并

production 生产环境，当testing环境通过测试后，将testing 合并到 主干 即成为生产环境的代码

另外我们可以通过release功能将主干的代码复制到tags中，命名采用版本号

创建配置文件

development

部署开发代码到开发环境

	cat deployment/conf/development/mydomain.com/www.conf
	
	REPOSITORY=git@192.168.2.1:mydomain.com/www.mydomain.com
	MODE=RSYNC
	OPTION="--delete --password-file=$PREFIX/conf/development/passwd"
	REMOTE="jszb@192.168.2.10"
	DESTINATION=mydomain.com/www.mydomain.com
			
创建密码文件

	$ cat deployment/conf/development/passwd
	eF9nJCcGKJPsiqZsfjGXxwfF41cLibTo
			
testing

部署测试分支到测试环境

	cat deployment/conf/testing/mydomain.com/www.conf
	
	REPOSITORY=git@192.168.2.1:mydomain.com/www.mydomain.com
	MODE=RSYNC
	OPTION="--delete --password-file=$PREFIX/conf/testing/passwd"
	REMOTE="jszb@192.168.2.10"
	DESTINATION=mydomain.com/www.mydomain.com
			
创建密码文件

	$ cat deployment/conf/testing/passwd
	eF9nJCcGKJPsiqZsfjGXxwfF41cLibTo
				
production

部署主干代码到远程主机

	cat deployment/conf/production/mydomain.com/www.conf
	
	REPOSITORY=git@192.168.2.1:mydomain.com/www.mydomain.com
	MODE=RSYNC
	OPTION="--delete --password-file=$PREFIX/conf/production/passwd"
	REMOTE="jszb@192.168.2.10"
	DESTINATION=mydomain.com/www.mydomain.com
			
创建密码文件

	$ cat deployment/conf/production/passwd
	eF9nJCcGKJPsiqZsfjGXxwfF41cLibTo
			
配置排出列表

有时我们不希望某些文件被上传到服务器上。我们可以通过排除列表来排除上传

	cat exclude/mydomain.com/www.lst
	/test/phpinfo.php
	/config/database.php
	/backup/*.sql
			
配置文件管理

生产环境的安全问题，例如数据库联接信息，开发环境与测试环境的数据库是可以供发人员和测试人员随意操作的，损坏之后恢复即可，但生产环境的数据库是不能随便操作的，除运维人员其他人是不应该有权限的， 我们希望部署到生产环境的时候使用另一个配置文件，并且这个配置文件只有运维人员才能编辑。

config/database.php 将覆盖原有的配置文件，然后上传到生产环境

vim share/production/mydomain.com/www/config/database.php
...
你的数据库连接信息
...
			
部署前/后脚本

部署前需要做什么

	$ cat libexec/mydomain.com/www/before
	rsync -au $DEPLOY_HOME/src/production/mydomain.com/www.mydomain.com/cn/* $DEPLOY_HOME/src/production/mydomain.com/www.mydomain.com/news/
	rsync -au $DEPLOY_HOME/src/production/mydomain.com/www.mydomain.com/images/* $DEPLOY_HOME/src/production/mydomain.com/www.mydomain.com/bbs/images/
	rsync -au $DEPLOY_HOME/src/production/mydomain.com/www.mydomain.com/css/* $DEPLOY_HOME/src/production/mydomain.com/www.mydomain.com/news/css
			
部署后需要做什么

	cat libexec/mydomain.com/www/after
	ssh www@192.168.1.1 "chown www:www -R /www/mydomain.com"
	ssh www@192.168.1.1 "chown 700 -R /www/mydomain.com"
	ssh www@192.168.1.1 "chown 777 -R /www/mydomain.com/www.mydomain.com/images/upload"
				
配置部署节点

在需要部署的节点上安装rsync

		
	yum install xinetd rsync -y
	
	vim /etc/xinetd.d/rsync <<VIM > /dev/null 2>&1
	:%s/yes/no/
	:wq
	VIM
	
	# service xinetd restart
	Stopping xinetd:                                           [  OK  ]
	Starting xinetd:                                           [  OK  ]
		
		
/etc/rsyncd.conf 配置文件

		
	# cat /etc/rsyncd.conf
	uid = root
	gid = root
	use chroot = no
	max connections = 8
	pid file = /var/run/rsyncd.pid
	lock file = /var/run/rsync.lock
	log file = /var/log/rsyncd.log
	
	hosts deny=*
	hosts allow=192.168.2.0/255.255.255.0
	
	[www]
	    uid = www
	    gid = www
	    path = /www
	    ignore errors
	    read only = no
	    list = no
	    auth users = www
	    secrets file = /etc/rsyncd.passwd
	[mydomain.com]
	    uid = www
	    gid = www
	    path = /www/mydomain.com
	    ignore errors
	    read only = no
	    list = no
	    auth users = mydomain
	    secrets file = /etc/rsyncd.passwd
	[example.com]
	    uid = www
	    gid = www
	    path = /www/example.com
	    ignore errors
	    read only = no
	    list = no
	    auth users = example
	    secrets file = /etc/rsyncd.passwd
			
		
创建密码

		
	cat > /etc/rsyncd.passwd <<EOD
	www:eF9nJCcGKJPsiqZsfjGXxwfF41cLibTo
	mydomain:eF9nJCcGKJPsiqZsfjGXxwfF41cLibTo
	example:eF9nJCcGKJPsiqZsfjGXxwfF41cLibTo
	EOD
			
		
部署代码

	Tip
development | testing 建议使用分支管理， 而production是用master分支

开发环境部署

	$ ~/deployment/run branch development mydomain.com www development 首次需要运行，切换到开发分支
	$ ~/deployment/run development mydomain.com www
		
测试环境部署

	$ ~/deployment/run branch development mydomain.com www testing 首次需要运行，切换到开发分支
	$ ~/deployment/run testing mydomain.com www
		
如果每个bug一个分支的情况可以每次先运行

	$ ~/deployment/run branch development mydomain.com www bug0005
		
生产环境部署

	$ ~/deployment/run production mydomain.com  www
		
每次部署都会在服务器 /www/mydomain.com/backup/ 下备份更改的文件

回撤操作

当程序升级失败需要立即回撤到指定版本时使用

			
	$ ~/deployment/run revert {development|testing|production} <domain> <host> <revision>
				
				
	./run revert development mydomain www 29dd5c3de6559e2ea6749f5a146ee36cbae750a7
	./run revert testing mydomain www 29dd5c3de6559e2ea6749f5a146ee36cbae750a7
	./run revert production mydomain www 29dd5c3de6559e2ea6749f5a146ee36cbae750a7
			
发行一个版本

release 升级你的版本

	$ ~/deployment/run release mydomain.com www stable-2.0
			
分支管理

查看当前分支

	[www@manager deployment]$ ./run branch development mydomain.com www
	* master
		
切换分支

	[www@manager deployment]$ ./run branch development mydomain.com www development
	HEAD is now at 461b796 提交最新代码
	Branch development set up to track remote branch development from origin.
	Switched to a new branch 'development'
		
现在已经切换到开发分支

	[www@manager deployment]$ ./run branch development mydomain.com www
	* development
	  master
		
备份操作

将生产环境备份至本地

	$ ~/deployment/run backup mydomain.com www /backup/2012-06-12/
		
日志

部署日志 deploy.YYYY-MM-DD.log, 记录部署时间与动态

	$ cat log/deploy.2012-08-03.log
	[2012-12-06 21:52:05] [update] /opt/git/testing/mydomain.com/m.mydomain.com
	[2012-12-06 21:52:10] [deploy] testing/mydomain.com/m.mydomain.com => www@192.168.2.15:mydomain.com/m.mydomain.com
	[2012-12-06 21:53:13] [checkout] commit:29dd5c3de6559e2ea6749f5a146ee36cbae750a7 /opt/git/testing/mydomain.com/m.mydomain.com
	[2012-12-06 21:53:18] [deploy] testing/mydomain.com/m.mydomain.com => www@192.168.2.15:mydomain.com/m.mydomain.com
	[2012-12-06 21:53:39] [update] /opt/git/testing/mydomain.com/m.mydomain.com
	[2012-12-06 21:53:45] [deploy] testing/mydomain.com/m.mydomain.com => www@192.168.2.15:mydomain.com/m.mydomain.com
	[2012-12-06 21:54:08] [update] /opt/git/testing/mydomain.com/m.mydomain.com
	[2012-12-06 21:54:10] [deploy] testing/mydomain.com/m.mydomain.com => www@192.168.2.15:mydomain.com/m.mydomain.com
	[2012-12-06 21:54:13] [checkout] commit:29dd5c3de6559e2ea6749f5a146ee36cbae750a7 /opt/git/testing/mydomain.com/m.mydomain.com
	[2012-12-06 21:54:15] [deploy] testing/mydomain.com/m.mydomain.com => www@192.168.2.15:mydomain.com/m.mydomain.com
			
项目日志 www.example.com.log 记录项目有哪些更新, 上传的细节, 你能通过日志看到那些文件被上传

		
	$ cat log/www.example.com.log
	--------------------------------------------------
	HEAD is now at 03b3ad5 XXXXXXXXXXXX
	- share:
	- libexec:
	2012/12/06 21:53:45 [12488] building file list
	2012/12/06 21:53:45 [12488] .d..t...... application/config/development/
	2012/12/06 21:53:45 [12488] <f.st...... application/config/development/database.php
	2012/12/06 21:53:45 [12488] .d..t...... application/controllers/
	2012/12/06 21:53:45 [12488] <f.st...... application/controllers/info.php
	2012/12/06 21:53:45 [12488] .d..t...... application/core/
	2012/12/06 21:53:45 [12488] <f.st...... application/core/MY_Controller.php
	2012/12/06 21:53:45 [12488] .d..t...... application/models/
	2012/12/06 21:53:45 [12488] <f.st...... application/models/news.php
	2012/12/06 21:53:45 [12488] .d..t...... application/views/
	2012/12/06 21:53:45 [12488] <f.st...... application/views/example.html
	2012/12/06 21:53:45 [12488] <f.st...... application/views/index.php
	2012/12/06 21:53:45 [12488] .d..t...... resources/css/
	2012/12/06 21:53:45 [12488] <f.st...... resources/css/m.css
	2012/12/06 21:53:45 [12488] sent 23640 bytes  received 421 bytes  3701.69 bytes/sec
	2012/12/06 21:53:45 [12488] total size is 2869760  speedup is 119.27
	--------------------------------------------------
			
			
debug

启用调试模式

	vim bin/deploy.git
	
	DEBUG=yes
		
然后查看log/debug.log
