# dmtsGithubBlog
我的github博客备份


### hexo + github
#### 1. hexo可以从github上hexo下载,里面包含了hexo等相关可执行文件
#### 2. git自行安装
#### 3. 发布新文章，然后同步到github上去

#### 注意1：从github上下载的theme主题文件夹，无法push到自己的github上去，因为其中包含了.git文件夹，解决方法：直接打开文件夹，图形界面拷贝一下，去掉.git等相关文件，然后同步；

#### 注意2: 用hexo init初始化后的文件夹为执行hexo g --> hexo d的目录，若拷贝到另外一个文件夹则，再次执行: hexo g --> hexo d，则发布博客之后，浏览器打开博客主页时，会显示乱码，无法正常显示，直接从github上clone一份也不行，只能在原始执行hexo init的目录进行博客发布；
