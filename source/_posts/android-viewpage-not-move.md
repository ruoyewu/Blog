---
title: Android ViewPager 设置不可滑动
date: 2018-2-28 11:20
tags:
	- android
---

在实际开发中有时会使用到使 ViewPager 不可滑动的要求，比如在 ViewPager 与它包含的某一页出现了滑动冲突，同时这个滑动冲突还不能依靠修改当前页中的某个控件来进行修复的，可以通过关闭 ViewPager 的手势滑动功能来消除滑动冲突。具体的代码如下：

```java
class CustomViewPager extends ViewPager {
    private boolean isPagingEnable = true;
    
    public CustomViewPager(Context context) {
        super(context);
    }
    
    public CustomViewPager(Context context, AttributeSet attrs)  {
        super(context, attrs);
    }
    
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        return isPagingEnable && super.onTouchEvent(ev);
    }
    
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return isPagingEnable && super.onInterceptTouchEvent(ev);
    }
    
    public void setPagingEnable(boolean enable) {
        isPagingEnable = enable;
    }
}
```

