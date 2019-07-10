### 博客地址
https://vevenlcf.github.io/blog/

### 必改内容

**1. swiftype**  此服务提供站内搜索功能

服务地址：https://swiftype.com/

设置方法可参考 http://opiece.me/2015/04/16/site-search-by-swiftype/

设置完毕后，您需要修改 _config.yml 中 swiftype.searchId。

在自己的引擎中，进入 Setup and integration -> Install Search, 你将找到 swiftype.searchId 

示例id为： 4s9rUZXGeG1eRBzqTUDA。

  <script type="text/javascript">
    (function(w,d,t,u,n,s,e){w['SwiftypeObject']=n;w[n]=w[n]||function(){
    (w[n].q=w[n].q||[]).push(arguments);};s=d.createElement(t);
    e=d.getElementsByTagName(t)[0];s.async=1;s.src=u;e.parentNode.insertBefore(s,e);
    })(window,document,'script','//s.swiftypecdn.com/install/v2/st.js','_st');

    _st('install','4s9rUZXGeG1eRBzqTUDA','2.0.0');
  </script>
  
可以参考 [搭建swiftype站内搜索的几点说明](https://www.jianshu.com/p/f2df26584e87)

**2. gitment** 此服务提供评论功能

服务地址：https://github.com/imsun/gitment

设置完毕后, 需要修改 _config.yml 中的 gitment。

可以参考： [Gitment：使用 GitHub Issues 搭建评论系统](https://imsun.net/posts/gitment-introduction/)
