---
title: github个人博客安装
catalog: true
p: blog\blog-install
date: 2019-09-19 11:37:55
subtitle:
header-img:
tags:
---
> Github个人博客安装  
# 一、安装博客
## 1.安装git、nodejs  
根据个人需求安装配置github及nodejs。
## 2.创建仓库
在github上创建一个以username.github.io为名的空仓库,其中username换成自己的用户名,不能是私有仓库，否则将无法访问。  
![github_create_repository](https://res.cloudinary.com/daxsyjegk/image/upload/v1568947385/E80F903E-0D21-4ea5-BB90-1B619B40D79B_ihaep7.png)
## 3.配置ssh key
通过配置ssh key要提交代码到github
```bash
$ cd ~/. ssh #检查本机已存在的ssh密钥
```
如果提示：No such file or directory 说明你是第一次使用git。
```bash
ssh-keygen -t rsa -C "邮件地址"
```
然后连续3次回车，最终会生成一个文件在用户目录下，打开用户目录，找到.ssh\id_rsa.pub文件，记事本打开并复制里面的内容，
打开你的github主页，进入个人设置 -> SSH and GPG keys -> New SSH key：
![ssh key add](https://res.cloudinary.com/daxsyjegk/image/upload/v1568948516/C910C253-690B-4503-AFAF-15D98A58D101_qlxvae.png)  
将刚复制的内容粘贴到key那里，title随便填，保存。
## 4.hexo安装及介绍  
### 4.1 hexo的安装
```bash
npm install -g hexo
```
在电脑的某个地方新建一个名为blog的文件夹（名字可以随便取），比如我的是C:\Workspaces\blog，由于这个文件夹将来就作为你存放代码的地方，所以最好不要随便放。
```
cd /f/Workspaces/hexo/
$ hexo init
```
hexo会自动下载一些文件到这个目录，包括node_modules。
### 4.2 hexo的目录
目录结构如下图：
![hexo dir](https://res.cloudinary.com/daxsyjegk/image/upload/v1568948955/074EF292-9711-4715-817E-7FB794A61887_fpuhvj.png)
其中各个目录的作用如下：  
.deploy_git: git部署用的文件。比如你写好的博客想部署到 GitHub Pages上去的话，可以用git部署插件，那个插件会创建这个目录  
node_modules: node.js用到的安装到当前“项目／目录”的插件／模块目录  
public: hexo编译之后的网站的目录  
.gitignore: git的配置文件，里面定义了不列入git管理的内容设置
_config.yml：hexo的配置文件  
### 4.3 hexo的配置文件_config.yml
在刚利用hexo搭建博客时，最重要的是将项目推送至github，故需先在_config.yml中配置这一属性。  
![github config](https://res.cloudinary.com/daxsyjegk/image/upload/v1568949666/59EE5AC5-8292-42f6-9630-B944052CAB72_zcctpg.png)  
因为是利用github pages来进行部署博客的，type自然填写git，repository填写在第一步中创建的仓库的ssh地址，
格式为git@github.com:username/username.github.io.git，分支branch为master。
### 4.4 hexo基础命令
```bash
hexo n == hexo new    #创建新的md文件
hexo g == hexo generate    #生成静态资源
hexo s == hexo server    #在本地启动
hexo d == hexo deploy    #上传至github
```
若需在创建md是对md进行分类，推荐使用一下命令
```bash
hexo n md_title -p md_category/md_name    #在_posts/d_category目录下创建以md_name为文件名的title为md_title的md  
```
---
# 二、多电脑更新博客  
实现多台电脑提交与更新github pages博客的关键在于将博客的文件夹上传至github进行同步与更新。  
本文的博客文件将提交至github pages的托管仓库，以不同的分支保存。
## 1.github操作
在github pages的托管仓库新建一个hexo分支，并在setting中将该分支设置为默认分支。  
在任意目录下新建一个文件夹，在该文件夹中打开git bash，执行以下命令:
```bash
git clone git@github.com:username/username.github.io.git
```
完成之后将文件夹中除.git目录以外的文件全部删除，在该目录下打开git bash，执行以下命令：
```bash
git add -A
git commit -m "--"
git push origin hexo
```
执行完以上命令之后，github上的hexo分支将被清空。
完成之后将.git文件夹拷贝至博客目录中，在推送之前先将.gitignore进行修改为以下内容：
```
.DS_Store
Thumbs.db
db.json
*.log
public/
.deploy*/
.idea/
node_modules/
```
完成之后再次执行以下命令：
```bash
git add -A
git commit -m "--"
git push origin hexo
```
至此，老电脑操作完成。
## 2.新电脑配置博客
新电脑按照第一节中的安装步骤安装安装完基本服务。
选好博客安装的目录， git clone git@github.com:username/username.github.io.git 。  
cd 到博客目录，npm install、hexo g && hexo s，安装依赖，生成和启动博客服务。正常的话，浏览器打开 localhost:4000 可以看到博客了。至此新电脑操作完毕。  
以后无论在哪台电脑上，更新以及提交博客，依次执行，git pull，git add -A ，git commit -m "--"，git push origin hexo，hexo clean && hexo g && hexo d 即可。
