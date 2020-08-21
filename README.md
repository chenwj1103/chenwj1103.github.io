# chenwj1103.github.io  chenwj.cn

### 新项目启动命令汇总
1. clone文件
- git clone https://github.com/chenwj1103/chenwj1103.github.io.git

2. 安装npm
- npm install

3. Hexo的安装
- npm install hexo --save

4. Hexo客户端的安装
- npm install -g hexo-cli

5. 安装hexo-util
- npm install -- save-dev hexo-util

6. 安装 hexo-server 
- npm install hexo-server --save

7. 本站点使用的是Local Search
- 安装hexo-generator-searchdb

npm install hexo-generator-searchdb --save

8. 安装hexo-deployer-git插件
npm install hexo-deployer-git --save
```

在站点myBlog/_config.yml中添加search字段，如下：
search:
 path: search.xml
 field: post
format: html
limit: 10000

```



### 项目发布命令

1. 清除缓存文件 (db.json) 和已生成的静态文件 (public)。在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，您可能需要运行该命令。
- hexo clean

2. 生成静态文件。
- hexo generate 
  -d, --deploy  文件生成后立即部署网站
  -w, --watch 监视文件变动

3. 启动服务器
- hexo server
  -p, --port  重设端口
  -s, --static  只使用静态文件
  -l, --log 启动日记记录，使用覆盖记录格式

4. 部署网站
- hexo deploy
  -g, --generate  部署之前预先生成静态文件

5. 调试
- hexo --debug


### 写作

1. 新建文章
- hexo new [layout] <title>

  新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 default_layout 参数代替。如果标题包含空格的话，请使用引号括起来。

2. 显示草稿
- hexo --draft


3. 新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站
- hexo init [folder]

 
https://liguanghe.github.io/2017/11/06/blogRebuilt/


