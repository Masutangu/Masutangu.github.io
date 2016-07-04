---
layout: post
date: 2016-07-04T08:06:35+08:00
title: matplotlib配置支持中文
category: 工作
---

这篇文章介绍linux下如何配置matplot来支持中文显示。

1. 确定系统已有哪些支持中文字体

    linux下运行 `fc-list :lang=zh` 命令会输出所有支持中文的字体。
    如果为空，可以按如下步骤添加：
    *   拷贝一份window `C:\windows\fonts\` 目录下的任一中文字体文件，例如 `MSYH.ttc` （微软雅黑）
    *   重命名为MSYH.ttf，并放到linux的 `/usr/share/fonts/chinese/`目录下 
    *   linux下执行 `fc-cache /usr/share/fonts/chinese` 清空字体缓存
    
2. 修改matplotlibrc配置
    *   打开python安装目录下的 `lib/site-packages/matplotlib/mpl-data/matplotlibrc` 文件，将 `font.family` 和 `font.sans-serif` 这两行开头的注释删掉，并在 `font.sans-serif` 这行添加 `Microsoft YaHei` (或系统支持的其他中文字体）
    *   删除 `~/.cache/matplotlib` 缓存目录

3. 修改python代码，指定使用中文字体

    ```
        from matplotlib import rcParams
        rcParams['font.sans-serif'] = ['Microsoft YaHei']  
    ``` 

Done!
    