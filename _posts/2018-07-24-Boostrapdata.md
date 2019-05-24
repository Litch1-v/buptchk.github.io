---
layout: post
title: Bootstrap中常见的data-属性
tags: Web开发

---
data-toggle，data-target,data-dismiss 相关

1. 关于data-*属性

   ``	data属性是HTML5允许开发者自由为其标签添加的属性，这种自定义属性一般以“data-”开头。

   ​	在Bootstrap中，可以仅仅通过调用data属性API就可以使用所有的Bootstrap插件。

   ​	Bootstrap中常见的data-属性：

    1. **data-toggle**

       data-toggle主要用于标签选择器，用于告诉JavaScript需要做什么eg：下拉菜单

       ```html
       <div class=”dropdown”>
         <a href=”#” class="dropdown-toggle" data-toggle="dropdown"></a>
         <ul class="dropdown-menu" role="menu" aria-labelledby="dLabel">
           ...
         </ul>
       </div>
       ```

       此处，data-toggle说明是一个下拉动作

       **常用的如下**：

       ```html
       data-toggle="dropdown"//下拉菜单
       data-toggle="model" //模态框事件
       data-toggle="tooltip"//提示框事件
       data-toggle="tab"//标签页
       data-toggle="collapse"//折叠
       data-toggle="popover"//弹出框
       data-toggle="button"//按钮事件
       ```

       

   	2. **data-target**

       data-target主要用于配合data-toggle使用。data-target主要用于配合data-toggle使用，常用于模态窗口、轮播图，作用是指定加载以及切换的目标。 

       eg：轮播图

       ```html
       <div id="myCarousel" class="carousel slide">
         <!-- 轮播（Carousel）指标 -->
           <ol class="carousel-indicators">
               <li data-target="#myCarousel" data-slide-to="0" class="active"></li>
               …
           </ol>  
        <!-- 轮播（Carousel）项目 -->
        <div class="carousel-inner">
        </div>
       </div>
       ```

   	3. **data-dismiss**

       常见的是在模态窗口中用于关闭模态窗口 data-dismiss=”modal” 

   	4. **data-slide**

       data-slide-to、data-slide、data-ride用于轮播图carousel

        属性 data-slide接受关键字 prev 或 next，用来改变幻灯片相对于当前位置的位置。 使用 data-slide-to 来向轮播传递一个原始滑动索引，data-slide-to="2"将把滑块移动到一个特定的索引，索引从 0开始计数。 data-ride="carousel" 属性用于标记轮播在页面加载时就开始动画播放 

       

       ****

