---
title: Hello CYL
---
封闭了大半年，喘过气来从新激活下这个hexo博客，感觉技术上的学习还是需要有一个地方记录下，尤其是工作中用不到的、自己学习的东西，也算是一种督促。

新的一年，在这里给自己订个目标，希望能在这个博客记录下这几个方面的学习和总结吧：

##### 函数式编程

之前学了一段时间scala后就拉下了，打算回头把scala99先解完，然后再继续学学scala的一些特性和常用库。有时间的话也许会看看SICP。

##### 响应式编程

对这个概念挺感兴趣的，大概了解了下感觉和FP很类似，也许会在工作中尝试一下Rx。

##### 机器学习

有时间的话在kaggle上耍耍。

---
最后附上搭建这个hexo博客的小记录吧，这个[博客](https://github.com/youlingman/youlingman.github.com)是放在Github User Pages上的hexo博客，使用了[Alex-fun](https://github.com/Alex-fun)的主题[jane](http://hejx.me/2016/02/07/hexo-theme-jane/)，搭建过程参考[这里](http://crazymilk.github.io/2015/12/28/GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2)，使用同一个repo的两条分支，分别存放hexo元配置文件和生成的博客静态文件，然后利用了Travis CI实现自动更新和发布，参考了[这里](https://xin053.github.io/2016/06/05/Travis%20CI%E8%87%AA%E5%8A%A8%E9%83%A8%E7%BD%B2Hexo%E5%8D%9A%E5%AE%A2%E5%88%B0Github/)，注意Github的personal access token只需要勾上pulic repo权限就可以完成推送，同时.travis.yml里的hexo deploy记得加上--silent选项，以避免在Travis CI的build log里暴露personal access token。
