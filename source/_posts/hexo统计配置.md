---
title: hexo统计配置
date: 2017-10-10 00:50:14
tags: hexo
categories: 博客

---


后续会整理hexo搭建博客的完整流程，包括过程中踩过的坑。

# 统计
1. [leancloud统计](https://www.jianshu.com/p/702a7aec4d00)
2. 卜算子统计 (/themes/next/_config.yml) 路径的themes的next的_config.xml文件中 enable改为true即可.

````
# Show PV/UV of the website/page with busuanzi.
# Get more information on http://ibruce.info/2015/04/04/busuanzi/
busuanzi_count:
  # count values only if the other configs are false
  enable: true
  # custom uv span for the whole site
  site_uv: true
  site_uv_header: <i class="fa fa-user"></i>
  site_uv_footer:
  # custom pv span for the whole site
  site_pv: true
  site_pv_header: <i class="fa fa-eye"></i>
  site_pv_footer:
  # custom pv span for one page only
  page_pv: true
  page_pv_header: <i class="fa fa-file-o"></i>
  page_pv_footer:
````
# 其它
1. [next主题秘籍](http://shenzekun.cn/hexo%E7%9A%84next%E4%B8%BB%E9%A2%98%E4%B8%AA%E6%80%A7%E5%8C%96%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.html)
2. [优化百度收录博客文章](http://www.yuan-ji.me/Hexo-%E4%BC%98%E5%8C%96%EF%BC%9A%E6%8F%90%E4%BA%A4sitemap%E5%8F%8A%E8%A7%A3%E5%86%B3%E7%99%BE%E5%BA%A6%E7%88%AC%E8%99%AB%E6%8A%93%E5%8F%96-GitHub-Pages-%E9%97%AE%E9%A2%98/)
3. [百度统计](https://tongji.baidu.com/web/24767668/homepage/index)
4. [daoVoice](http://dashboard.daovoice.io/app/7e36aff1/users?segment=all-users)
5. [来必力](https://livere.com/insight)
6. [SSR客户端](https://github.com/the0demiurge/CharlesScripts/blob/master/charles/bin/ssr)
6. [SSR客户端2](https://www.djangoz.com/2017/08/16/linux_setup_ssr/)