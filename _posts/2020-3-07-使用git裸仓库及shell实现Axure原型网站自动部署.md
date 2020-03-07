---
layout:     post
title:      使用git裸仓库及shell实现Axure原型网站自动部署
subtitle:   解决更新仓库时运行的shell脚本本身在更新范围内的问题
date:       2020-03-07
author:     gaopq
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Git
    - 裸仓库
    - Shell实战
---

### 技术栈
- Git
    - 构建裸仓库用于存放钩子/文件仓库
- Shell		
    - 用于更新项目原型版本及提供给不熟悉git操作的产品同事更新原型
- HTML		
    - 用于根据js配置文件展示项目原型及版本
- Js			
    - 用于存放所有项目的版本配置, 供html及shell脚本解析调用
- Nginx		
    - HTTP服务器用于配置web域名访问及简单账户密码限制
- AxureRP		
    - 产品同学用于生成用于原型预览的HTML文件
***
### 中央仓库
用于存放钩子，相当于中央仓库
```
cd /data/bare_repo
git init --bare rp.git
cd rp.git/hooks
vim post-receive
--- contents of post-receive ----
#!/bin/bash

unset GIT_DIR
dep_path="/data/prototype/"

cd "$dep_path"
git add . -A && git stash
git pull origin master

exit 0
--- contents end ----
```
### 本地仓库
用于存放供web访问的文件
```
cd /data/prototype
git init
git remote add origin /data/bare_repo/rp.git
```
***
### 项目结构
* config
    -  用于存放项目配置
* rp
    - 用于存放项目原型 HTML 文件
* scripts
    - 存放提供给产品同学执行的 shell 脚本，以及当前待运行脚本的隐藏备份文件
* index.html/version.html
    - web页面入口，用于页面展示所有产品的原型及版本

### 代码样例
**config/properties.js**
```
var rpVersion = {
  mmb: [0.1, 0.2],
  aitu: [0.1],
  wedis: [0.1, '0.2'],
  kqmp : [0.1, 0.2, 'web'],
  ddysApp: [2.13, 2.14],
  ddysWx: [2.13, 2.14],
  ddysPc: [2.13],
  ddysAdmin: [2.13],
  ddysXbld: [2.13]
}
``` 

**scripts/git_rp_repo_sync.sh**
``` 
#!/bin/bash
# encoding: utf-8
###################################################
# Author:                 gaopq
# Version:                0.03
# Email:                  peiqianggao@gmail.com
# ChangeLog:              add versions info for projects
# ChangeLog:              fixed bugs when this script updated itself and other user execute it before pull and update it.
# Description:            更新原型系统请先在 rp 文件夹下对应项目文件夹下创建代表版本的文件夹, 并把原型HTML文件生成到该文件夹下再执行本脚本
###################################################
CONF_FILE=$(dirname $0)/../config/properties.js
RP_DIR="$(dirname $0)/../"
TMP_SHELL_NAME=".tmp.sh"

# 获取所有项目名
get_projects(){
   while read line
   do
       if [[ $line != ${line%:*} ]]
       then
           projs="$projs ${line%:*}"
       fi
   done < $CONF_FILE
   echo $projs
}

# 获取某个项目的现有版本号, param: project_name
get_versions(){
    [[ $# -ne 1 ]] && return 1
    proj=$1
    versions=$(grep -w $proj $CONF_FILE | awk -F: '{print $2}')
    echo $versions | sed "s/ //g;s/[]\[,]/ /g;s/'//g;"
}

# 给某个项目添加一个版本, params: project_name, new_version
add_version(){
    [[ $# -ne 2 ]] && return 1
    proj=$1
    new_version=$2
    versions_pro=$(get_versions $proj)
    echo $versions_pro | grep -w $new_version > /dev/null
    if [[ $? -eq 0 ]]
    then
        return 1
    else
    # write new_version to file
    # 此处不能使用单引号
    # sed 's/(^ *$proj.*)(],.*$)/\1, ${new_version}\2/' $CONF_FILE
        sed -i "s/\(^ *$proj.*\)\(],.*$\)/\1, '${new_version}'\2/" $CONF_FILE
        return 0
    fi
}

# 通过 git 上传文件
rp_git_sync(){
    git add -A :/
    git commit -m "$(date +'%Y-%m-%d %H:%M:%S')[INFO]update project: $1, version: $2, From: $(uname -a)"
    echo -e "\e[1;32m][INFO]Waiting for updating and pushing ...\e[0m"
    git push origin master && echo -e "\e[1;32m$(date +'%Y-%m-%d %H:%M:%S') updated successfully.\e[0m"
}

# 从仓库更新文件并判断本shell文件是否在被更新的列表中
git_pull_test_shell_script_if_updated(){
    # shell_name=$(basename $0)
    # if the current running shell is not the $TMP_SHELLL_NAME
    [[ ${0%%$TMP_SHELL_NAME} == $0 ]] && cp -rf $0 $(dirname $0)/${TMP_SHELL_NAME}

    cd $RP_DIR
    git pull origin master
    [[ $? -ne 0 ]] && exec bash scripts/${TMP_SHELL_NAME}
}

# 开始工作
git_pull_test_shell_script_if_updated
echo -e "\e[1;32m\nPlease select project (select one and update all ^_^)\e[0m"
select project in $(get_projects)
do
    # vers=($(get_versions $project))
    # last_ver=${vers[((${#vers[@]}-1))]}
    last_ver=$(ls -t ${RP_DIR}rp/$project | head -1)
    echo "Versions: $(get_versions $project)"
    echo -en "\e[1;32m\nPlease input your version name (defaults: $last_ver): \e[0m"
    read version
    version=${version// /}
    version=${version:-$last_ver}
    add_version $project $version
    rp_git_sync $project $version
    break
done
exit 0
```

### `Nginx` 配置
```
server {
        listen       80;
        server_name  my_domin_name;
        root /data/prototype/;
        index index.html;
        access_log /data/logs/nginx/my_domain_name.log main;
        location ^~ / {
            try_files $uri $uri/ /index.html?$args;
			# 生成简单授权账号密码: htpasswd -c $NGINX_PATH/passwd_db/passwd.db username
            auth_basic "Input Password";
            auth_basic_user_file $NGINX_PATH/passwd_db/passwd.db;
        }

        location ^~ /scripts {
            deny all;
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico|html)$ {
            if (-f $request_filename) {
                root /data/prototype;
                expires 10s;
                }
        }
}
```
***
### 问题解决
* properties.js 
    > 1. 文件中版本号需要使用单引号或双引号括起，便于 `js` 解析
* 用户在执行本 `shell` 是发现本 `shell` 文件在远程仓库被修改， `git pull` 的时候报错
	> 1. 每次执行脚本前若发现执行的不是备份版，则备份一次本脚本
	> 2. 发现 `git pull` 失败时默认认为是本 `shell` 在待更新的数据中，则使用 `exec` 执行备份的脚本
### 用户端配置

```
# server 端先配置一个可以访问 /data/bare_repo 和 /data/prototype 的用户
# client 端使用 ssh-keygen 命令生成公钥私钥并配置到 server 上
git init
git remote add rp_user@ip:/data/bare_repo/rp.git
# 首次更新，获取初始化文件及 shell 脚本
git pull origin master 

# 使用: 
# 在 rp 文件夹下对应项目文件夹下创建代表版本的文件夹, 并把原型HTML文件生成到该文件夹下
# 直接将 scripts 下的 git_rp_repo_sync.sh 拖到 git-bash 或 terminal 中执行即可
# 可能需要: chmod +x git_rp_repo_sync.sh
```
***
### 效果预览
**Web**

![rp_index.png](http://upload-images.jianshu.io/upload_images/209514-bcc4b9071c6df911.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![rp_versions.png](http://upload-images.jianshu.io/upload_images/209514-7a76c1451b984631.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**客户端 `shell` 执行效果**

![rp_shell.png](http://upload-images.jianshu.io/upload_images/209514-2d4dfddf72734207.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
