#jenkins
一：jenkins自动化集成和部署前端项目
登陆远程服务器地址
1：安装node环境
  1.1：安装nvm
    wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
  1.2：添加node环境变量
    vi ~/.zshrc
    # 文件末尾添加
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
    [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
2：安装 Node
  2.1：# 查看当前存在的所有Node版本
    nvm list-remote
  2.2：选择需要的node版本
    nvm install v11.12.0
3:安装pm2 （需要Node服务支持的可以选择安装）
  3.1：npm install pm2 -g
4：下载 java 依赖包 和 git
  4.1:yum install java
  4.2:yum install git
  4.3:添加 Jenkins 源
    wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo
    rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
    yum install jenkins // 完成之后直接使用 yum 命令安装 Jenkins
5:配置jenkins
  5.1：修改jenkins权限
    vim /etc/sysconfig/jenkins
    找到$JENKINS_USER 改为 “root”:
    也可以修改jenkins服务启动的端口号为其他的。
  5.2：然后更改执行以下命令 Jenkins home，webroot 和日志的所有权：
    chown -R root:root /var/lib/jenkins
    chown -R root:root /var/cache/jenkins
    chown -R root:root /var/log/jenkins
6:启动jenkins
  6.1：service jenkins restart
7：浏览器输入服务ip地址和jenkins服务的端口号（默认为8080）
8：输入解锁jenkins的管理员密码
  8.1：到远程服务器下输入 vim /var/lib/jenkins/secrets/initialAdminPassword 获取密码串输入
9：选择安装默认插件
10：创建管理员用户配置管理员用户名和密码
11:Jenkins 配置 Node 环境（项目使用了Node环境，所以需要在 Jenkins 中配置我们打包需要的 Node 环境。）
  11.1:jenkins安装NodeJS 插件
  11.2：打开 系统管理 => 全局工具配置 选中 NodeJS 栏， 点击 NodeJs 安装 按钮。
  11.3：选自动安装，选择一个使用的Node版本（node_11.12.0）。
12:到git项目下生成token（码云或github都可以，推荐码云git在阿里云上可能会被墙掉需要另外客气代理才能使用）我这里以码云为例配置
13：实现git钩子功能
  13.1：打开刚创建的任务，选择配置，点击源码管理tab
        添加远程仓库地址，配置登录名及密码及分支。
  13.2：安装Generic Webhook Trigger Plugin插件（系统管理-插件管理-搜索Generic Webhook Trigger Plugin）如果可选插件列表为空，点         击高级标签页，替换升级站点的URL为：http://mirror.xmission.com/jenkins/updates/update-center.json并且点击提交和立即获           取。
  13.3：点击构建触发器tab,添加触发器,勾选触发器
        第2步安装的触发器插件功能很强大，可以根据不同的触发参数触发不同的构建操作，比如我向远程仓库提交的是master分支的代码，就执行         代码部署工作，我向远程仓库提交的是某个feature分支，就执行单元测试，单元测试通过后合并至dev分支。灵活性很高，可以自定义配置适         合自己公司的方案，
  13.4：仓库配置钩子
        以码云为例，因为公司用的是码云，github的配置基本一致，进入码云项目主页后，点击管理-webhooks-添加
        输入的url地址为：http://<User ID>:<API Token>@<Jenkins IP地址>:端口/generic-webhook-trigger/invoke
        userid和api token在jenkins的系统管理-管理用户-admin-设置里
        Jenkins IP地址和端口是部署jenkins服务器的ip地址，端口号没改过的话就是8080。密码填你和上面userid对应的密码，我这里是root。
  13.5：测试钩子
        试下本地提交代码，提交代码后，jenkins也会开始一个任务,目前我们没有配置任务开始后让它做什么，所以默认它只会在你提交新代码后，         将新代码拉取到jenkins服务器上。到此为止，git钩子我们配置完成。
14：实现自动化构建
  14.1：jenkins里面配置node的环境
        首先，和本地运行npm script一样，我们要想在jenkins里面执行npm命令，先要在jenkins里面配置node的环境，可以通过配置环境变量的          方式引入node，也可以通过安装插件的方式，这里使用了插件的方式，安装一下nvm wrapper这个插件。
  14.2：打开刚刚的jenkins任务，点击配置里面的构建环境，勾选“run the bulid in an NVM managed Environment”，并指定一个node版本。
  14.3:点击构建，把要执行的命令输进去
       #!/bin/bash
       # 输出当前环境
       echo '开始项目构建命令'
       echo $PATH
       node -v
       npm -v
       echo '当前分支'
       git branch
       # 切换拉取代码
       echo '拉取代码'
       git pull origin master
       # 安装依赖库
       echo '安装依赖库'
       npm install
       # 执行打包
       echo '开始打包'
       npm run build
       echo '打包完成'
       echo $PATH
15：实现自动化部署，配置ssh服务
  15.1:安装Publish over SSH插件
  15.2：服务器端生成公钥和私钥（linux生成公钥私钥参考地址：https://www.cnblogs.com/wangqianqiannb/p/7200791.html?         utm_source=itdadao&utm_medium=referral）
  15.2：打开 系统管理 => 系统设置 ，选择 Publish over SSH 项，填写相应的名字，Path to key（私钥地址）服务器地址，用户名，跳转目录   等
  15.3：配置完成后，点击Test Configuration测试一下是否可以连接上，如果成功会返回success，失败会返回报错信息，根据报错信息改正即可。
  15.4：点击应用 后保存
16：点击构建后操作，增加构建后操作步骤
  16.1:选择send build artificial over SSH
  16.2:Source files
       zip/test.zip
  16.3:Remove prefix
       zip
  16.4:	Exec command
       #!/bin/bash
       cd /usr/share/nginx/html/html90/
       rm -rf dist.tar.gz
       rm -rf index.html
       rm -rf static
       unzip test.zip
       rm -rf test.zip
  16.5:点击应用 后保存
17：本地push代码 服务器会自动构建个部署到其他服务
 
 
 参考文章
 1：实战笔记：Jenkins打造强大的前端自动化工作流(https://juejin.im/post/5ad1980e6fb9a028c42ea1be#heading-8)
 2:前端er，Jenkins持续化集成Webpack项目(https://juejin.im/post/5c9dc0de6fb9a071105deeb2#heading-16)
 3:Jenkins +nginx 搭建前端构建环境(https://juejin.im/post/5b371678f265da599f68dfa2)
 4:jenkins远程连接linux配置测试(https://www.cnblogs.com/wangqianqiannb/p/7200791.html?utm_source=itdadao&utm_medium=referral)
 5:前端必会的 Nginx入门教程(https://juejin.im/post/5bd7a6046fb9a05d2c43f8c7)
