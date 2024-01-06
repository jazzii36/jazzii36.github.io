## docsify push github page<!-- {docsify-ignore} -->

### 问题描述

1. 当我们用docsify试图发布到github.io时，docsify 项目特殊文件，如`_sidebar.md`,`_navbar.md`规范要求以`_ `开头 。
2. github page默认generate引擎是jekyll，整个jekyll默认会将`_`开头的文件忽视掉，导致所有页面都会没有侧边导航栏。

### 解决办法

1. 网上做法是项目中添加`.nojekyll`空文件，注意不要被git ignore，实测无效。
2. 推荐我的解决办法：
   - 修改_sidebar,_navbar,去掉下划线
   - 修改docsfiy中的index.html:
        ```
        window.$docsify = {
      executeScript: true,
      loadSidebar: 'sidebar.md',
      loadNavbar:'navbar.md'
     ...
        ```
   - 把`loadNavbar:true`修改为`loadSidebar: 'sidebar.md'`,注意值为项目文件的相对index.html所在目录路径。