---
title: Mac 使用手册
date: 2017-06-08 00:00:00
tags: 
	- mac
	- 工具
---

# Mac使用手册

1.  鼠标滑动速度修改

    1.  鼠标滑动速度

        ```
        defaults read -g com.apple.mouse.scaling
        ```

        得到当前鼠标滚动速度值

        ```
        defaults write -g com.apple.mouse.scaling $num
        ```

        `$num`为需要设置的值，值越大鼠标滑动越快

    2. 鼠标滚动速度

        ```
        defaults read -g com.apple.scrollwheel.scaling
        ```

        同上述

    3. 鼠标双击阈值

        ```
        defaults read -g com.apple.mouse.doubleClickThreshold
        ```

        同上述

    另外，以上参数修改后需要重起生效。