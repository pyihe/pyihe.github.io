---
layout: post
title: 'Github Pages-Jekyll H2O主题添加gitalk'
date: 2020-06-05
author: pyihe
tags: [学习]
---

## Gitalk介绍

[gitalk](https://github.com/gitalk/gitalk)是一个通过为每篇文章创建issue作为评论的评论插件，还有另外一个选择便是[gitment](https://github.com/imsun/gitment)，但gitment已经很久没维护了，所以大部分人选择了仍在维护的gitalk.

## 添加前的准备

添加gitalk前需要前往[GitHub]()注册账号（如果没有的话），并申请一个OAuth App，申请流程如下：
1. 前往Github Setting界面，选择Developer Settings

    ![](/assets/img/2020-06-05/setting.jpg)

2. 在OAuth Apps选项中点击New OAuth App，进入如下界面：

    ![](/assets/img/2020-06-05/oauth_app.jpg)
    
    * Application name: 应用的名字，可以随便取（其他人想要评论的话，需要登录Github授权，授权的时候显示的就是这里的Application name）
    * Homepage URL: 应用主页的完整URL，如: https://pyihe.github.io/
    * Application description: 应用描述，选填项
    * Authorization callback URL: 登录授权后回调的页面，直接填成与Homepage URL一样
    
注册完毕后会生成应用的`Client ID`和`Client Secret`，这两项在添加gitalk的过程中需要用到

## 开始添加gitalk

1. 在HomePages的配置文件`_config.yml`中的`comments`下面添加gitalk的配置信息，如下图所示: 
    
    ![](/assets/img/2020-06-05/config.jpg)

2. 修改`_layouts/`下的post.html文件，分三步:
    * 在html文件开头的`<html>`标签下添加如下两行代码，添加的代码可以从gitalk项目主页中拷贝
        ```html
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.css">
        <script src="https://unpkg.com/gitalk@latest/dist/gitalk.min.js"></script>
        ```
   
   * 在与`</header>`下面的标签中最外层`<div>`平级的地方添加如下代码，添加位置如下图所示: 
        ```html
        <div>
             <!--实际添加的时候请将\和'去掉-->
             \{ % 'if site.comments.gitalk' % \}
             <section class="post-footer-item comment">
             <div id="gitalk_container"></div>
             <div id="disqus_thread"></div>
             </section>
             \{ % 'endif' % \}
         </div>
        ```
        ![](/assets/img/2020-06-05/div.jpg)
   
   * 在post.html最后添加如下代码，添加位置如图所示: 
        ```html
        {% if site.comments.gitalk %}
          <script>
            var gitalk = new Gitalk({
              id: location.pathname,
              clientID: '\{\{ site.comments.clientID \}\}',  //这里其实可以直接填值，但是考虑到页面安全性，还是通过配置的方式添加
              clientSecret: '\{\{ site.comments.secret \}\}', 
              repo: '\{\{ site.comments.repo \}\}',  //\为转移符号，实际添加的时候请将其去掉
              owner: '\{\{ site.comments.owner \}\}',
              admin: '\{\{ site.comments.admin \}\}',
              distractionFreeMode: '\{\{ site.comments.distractionFreeMode \}\}'
            })
        
            gitalk.render('disqus_thread')
          </script>
          {% endif %}
        ```
        ![](/assets/img/2020-06-05/js.jpg)
        
完成以上两步，正常情况下Github Pages就已经添加gitalk成功了。但是事与愿违，过程中总会遇到各种问题，导致结果没有达到预期，下面是我在添加过程中遇到的问题

## 过程中遇到的问题

1. 评论框上方出现`gitalk Error: Validation Failed.`的错误字样，如下图：

    ![](/assets/img/2020-06-05/QA1.jpg)
    
    * 原因: 因为最后添加代码时，`id: location.pathname,`中的id字段超长（不能超过50）
    * 解决: 将id字段的值改为: `id: '[[page.date]]'`，这样便可解决（更改的时候需要将两个[]替换为两个花括号{}）。
    * 另一种解决办法是通过MD5 hash`location.pathname`。网上下载`md5.min.js`文件，放入`/assets/js/`下（具体目录视具体情况而定，反正是js目录），然后在`_layouts/post.html`文件中引入，如下图：
        
        ![](/assets/img/2020-06-05/md5.jpg)
      
      然后将id字段改为: `id: md5(location.pathname)`即可
      
2. 添加了gitalk.css后，导致文章中原本正常的代码格式发生了变化，如下图: 
    
    ![](/assets/img/2020-06-05/code_css.jpg)
    
   * 原因: 因为不懂前端，只能猜测是引入的css跟自己的css发生了冲突
   * 解决: 在浏览器中打开链接`https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.css`并搜索`.gt-container`，然后在css目录下新建文件`gitalk.css`，并将刚刚搜索结果中的高亮部分全部拷贝到新建的`gitalk.css`文件中，然后在post_html开头将`<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.css">`替换成`<link rel="stylesheet" href="{{site.baseurl}}/assets/css/gitalk.css}">`，具体路径需要自己确定。修复后如下图：
   
   ![](/assets/img/2020-06-05/code.jpg)
   
3. 文章下方的作者头像和gitalk文字重叠，如下图：
    
    ![](/assets/img/2020-06-05/head.jpg)
    
    * 原因: html代码中gitalk section样式问题
    * 解决: 添加style，代码如下，
        ```html
        <div>
            {% if site.comments.gitalk %}
            <section class="post-footer-item comment" style="padding-bottom: 4em">
              <div id="gitalk_container"></div>
              <div id="disqus_thread"></div>
            </section>
            {% endif %}
          </div>
        ```
4. `GET https://api.github.com/user 401 (Unauthorized)`(或者评论框上方出现`Error: Network Error`)
    * 解决: Unauthorized是因为没登录Github账号的原因，登录过后，Error: Network Error也消失了
    
5. 在chorme浏览器console中出现`Unchecked runtime.lastError: The message port closed before a response was received.`
    * 原因: 不得而知，大概是浏览器第三方扩展程序的原因
    * 解决: 挨个禁用你的扩展程序，然后刷新页面查看该错误是否消失，如果消失，便是该插件导致的。另一种做法是不禁用，改为点击时才允许访问，如下图：
    
        ![](/assets/img/2020-06-05/plug-in.jpg)

## 希望能帮助到你。