---
title: Android 事件分发
date: 2017-10-19 00:00:00
tags: android
---

# android事件分发

## 责任链模式

有一公司A，一客户给公司的boss发派了一个任务，然后boss比较忙，旨在锻炼下属，就把任务抛给了下属，即二级boss，然后这个二级boss也与大boss有同样的想法，就再把这个任务抛给了下一级，一层一层，直到传递到最后一层了，一个刚入职的萌新，试用期还没过，一看到这个任务就头皮发麻，告诉自己的上级说自己做不了，没办法，这个上级决定自己做，无奈自己也不行，只能再告诉自己的上级，一层一层，最后任务原原本本的回到了大boss的手上，一边抱怨着现在的年轻人不行了，一边把这个任务自己完成。

之后，当这个客户再有其他什么任务要给boss的时候，如果上一个任务交给了boss底下的某个人C完成，那么大boss就对二级boss说，把这个任务还给C做吧，然后把这个任务又抛给了二级boss，二级boss再这样一层一层，最后到了C的手里，他二话不说，拿起就做，也懒得给下面的人了。

之后，这客户又有新任务了，然后把任务给了大boss，大boss依旧如上一个任务，要把任务一层一层传给C去做，不过当任务到了二级boss手里的时候，二级boss一看这个任务的内容，就生出了一种感觉，一种这个任务必须要由自己完成使命，于是他不顾大boss的吩咐，自己做了这个任务，然后他又告诉C说，以后这个客户的任务不需要C做了，C说哦，说完把所有关于这个客户之前任务的资料都删掉了。

之后，客户又有新任务了，大boss什么话都没说，就把资料扔给了二级boss，二级boss一看客户的名字，按理说这个任务应该是要给C的，但是上次自己把他的任务拦下来了，这次C还会老老实实的做吗？然后二级boss当机立断，自己很快地做完，交给了大boss，也是什么也没说。

故事本应该就这样结束了，可是二级boss把自己的工作拦截下来的事情被C发现了，C就很不爽了，他就立马告诉自己的上级说，“你跟上面说说，下次再有这个客户的任务，还是给我做吧，毕竟我一直在做”，然后一层一层，大boss就知道了。直到下一次此客户再发派任务的时候，大boss拿起笔记本一看，C要做这个，就告诉二级boss，把这个给C做去吧，然后潇洒走开。这下二级boss可不能再任由自己了，全公司上上下下都看着呢，就是自己再怎么隐蔽，也不能瞒得了所有人，还是老老实实按大boss说的做吧，于是C这下终于可以如愿以偿了。

使用了这种工作模式的公司A的生意越做越大了，很显然这样的工作模式能够很大程度上发挥每个人的工作效率，同时大boss也在之后的工作中不断完善自己公司的工作模式，渐臻完美。这就是责任链模式的逐层传递。

## 在android中

在 Android 的日常开发中，有一个比较大的模块叫做自定义 view ，自定义 view 又包括大致两个方面，自定义 ViewGroup 和自定义的 View，其中 ViewGroup 的自定义主要涉及的就是父 view 与子 view 之间的协调工作，包括对子 view 的位置、大小的控制，以及功能上的分配。而功能的分配，大多数情况下就是对用户输入事件的处理，因为现在的 android 以及其他平台的手机，大都是只有触摸屏而没有其他供用户与手机交互的方式，所以应用就应该能够根据用户的触摸情况，以及自身的逻辑，给予用户好的使用体验，什么时候用户是想与自己交互，什么时候用户是在与自己的子 view 交互，一个好的 viewGroup 都应该清楚这些。就像是上面的例子中，在某一层的人员接到一个任务时，他要知道怎么处理这个任务，给下一级，给自己，还是返回给上一级。

在 android 的 view 自定义中一般使用的关于事件分发的函数有这三个：

|          函数名          |                    说明                    | 返回结果                                     |
| :-------------------: | :--------------------------------------: | :--------------------------------------- |
|  dispatchTouchEvent   | 用来分发一个事件，当有事件传递进来的时候调用，判断是否要继续往下传递或者是自己消费 | true：消费此次事件，并且后续事件到这个函数就会终止传递<br />false：不消费此次事件，但事件终止传递，后续事件仍继续传递<br />super：调用后面两个函数的返回值 |
| onInterceptTouchEvent | 在 dispatchTouchEvent 内部调用，判断是否拦截此事件，只在 ViewGroup 中有这个方法 | true：表示拦截事件，调用 onTouchEvent 方法<br />false：不拦截，事件继续向下传递 |
|     onTouchEvent      |     在 dispatchTouchEvent 内部调用，处理点击事件     | true：消费事件，后续事件会直接传递到这里，不再判断是否拦截<br />false：将事件向上传递给 onTouchEvent 进行判读 |

先列出一张图：

![事件分发流程图](http://upload-images.jianshu.io/upload_images/966283-d01a5845f7426097.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>   看上面这张图，当用户产生一个触摸事件之后，最先会交给当前 activity 的 dispatchTouchEvent 处理，然后在 activity 的 dispatchTouchEvent 中会调用当前布局 window 的 dispatchTouchEvent 方法，即 phoneWindow，
>
>   ```java
>   	/**
>        * Called to process touch screen events.  You can override this to
>        * intercept all touch screen events before they are dispatched to the
>        * window.  Be sure to call this implementation for touch screen events
>        * that should be handled normally.
>        *
>        * @param ev The touch screen event.
>        *
>        * @return boolean Return true if this event was consumed.
>        */
>       public boolean dispatchTouchEvent(MotionEvent ev) {
>           if (ev.getAction() == MotionEvent.ACTION_DOWN) {
>               onUserInteraction();
>           }
>           if (getWindow().superDispatchTouchEvent(ev)) {
>               return true;
>           }
>           return onTouchEvent(ev);
>       }
>   ```
>
>   而在phoneWindow 中的 dispatchTouchEvent 方法中又继续调用了 DecorView 的 superDispatchTouchEvent 方法。
>
>   ```java
>   	@Override
>   	public boolean superDispatchTouchEvent(MotionEvent event) {
>       	return mDecor.superDispatchTouchEvent(event);
>   	}
>   ```
>
>   然后在 DecorView 中，superDispatchTouchEvent 又而 DecorView 是一个 activity 的布局中最底层的一个 ViewGroup，从这里开始，这个触摸事件正式进入了 View 体系中进行分配。而在 DecorView 中，
>
>   ```java
>   	public boolean superDispatchTouchEvent(MotionEvent event) {
>           return super.dispatchTouchEvent(event);
>       }
>   ```
>
>   又继续调用了父类的 dispatchTouchEvent 方法，DecorView 的父类是一个 FrameLayout，FrameLayout 并没有关于dispatchTouchEvent 的实现，接着找 FrameLayout 的父类 ViewGroup，在 ViewGroup 中有对 dispatchTouchEvent 方法的具体实现，包括各种对于是否拦截此次事件的判断。
>
>   另外，在 activity 的 dispatchTouchEvent 方法中，有一个判断，`if ( getWindow().superDispatchTouchEvent(ev) )`，这里面的返回值就是各级 view 的 dispatchTouchEvent 的返回值，表示是否消费事件，所以这里的逻辑就是，判断是否有 view 消费了这次事件，如果有，直接返回 true，如果没有，就把这个事件交给自己的 onTouchEvent 处理，并返回处理的结果。

### dispatchTouchEvent

当有一个新的事件产生的时候，会首先调用一个 view 的 dispatchTouchEvent 对这个事件进行分发，dispatchTouchEvent 的返回值表示是否消费这个事件，在函数内部用 `handled` 这个值表示。在一个简单的系列事件中，一般包含一个 MotionEvent.ACTION_DOWN，若干个（可为零） MotionEvent.ACTION_MOVE，一个 MotionEvent.ACTION_UP 或 MotionEvent.ACTION_CANCEL。其中 MotionEvent.ACTION_DOWN 被认为是一个系列事件的开始的标志，也决定着后续事件要在何处处理。一般情况下，如果 MotionEvent.ACTION_DOWN 被某个 view A 消费了，那么这个系列事件的后续事件，在传送到 A 的时候就会直接调用 onTouchEvent，而不会再考虑是否要拦截，以及是否分发给子 view 了。所以在遇到一个 MotionEvent.ACTION_DOWN 事件的时候，就意味着一个新的系列事件开始了，所以 view 要首先清除之前的事件分发的时候的残留数据，重新开始新的旅途，大有从头来过的意思。然后就是调用自己的 onInterceptTouchEvent 来对这个事件进行判断。不过在调用这个方法之前还要进行一次判断，即：

```java
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}
```

这里只有在 disallowIntercept 为 false 的时候才会调用 onInterceptTouchEvent 进行是否拦截判断，那么这个 disallowIntercept 的意思就是是否允许拦截，那么这个允许拦截是谁允许的？是当前 ViewGroup 的子 view，默认这个值是 false，只有在子 view 里面调用了 `parent.requestDisallowInterceptTouchEvent(disallowIntercept)`的时候就会把这个值赋为 true，表示子 view 需要这个事件，而自己不能拦截。这里的 parent 可以由 `getParent()`获得，getParent 返回的是一个 ViewParent 实例，ViewGroup 实现了 ViewParent，可以看一下 ViewGroup 中的 requestDisallowInterceptTouchEvent 方法：

```java
@Override
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        // We're already in this state, assume our ancestors are too
        return;
    }
    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }
    // Pass it up to our parent
    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```

可以看到当自己的子 view 调用 parent.requestDisallowInterceptTouchEvent() 方法的时候，ViewGroup 就会将自己的 FLAG_DISALLOW_INTERCEPT 置为一个值，使得`final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;`为 true，同时当前 ViewGroup 还会继续调用自己的父类的此方法，保证这一系列事件能够到达需要这个事件的 view 那里。同时在判断是否拦截的时候除了判断是否是 MotionEvent.ACTION_DOWN 之外，还判断了 mFirstTouchTarget 是否为空

//TODO

之后会在继续判断是否被取消了此次事件，即是否本应该传递到这里的一个事件，被上层 view 给拦截了，那么就会在这里响应一个 MotionEvent.ACTION_CANCEL，如：‘

```java
// Check for cancelation.
final boolean canceled = resetCancelNextUpFlag(this)
        || actionMasked == MotionEvent.ACTION_CANCEL;
```

如果这个值为 true，表示此次事件被拦截了，那么就不能再继续往下传递了。如果当前事件没有被自己的上层 view 拦截，同时也没有被自己拦截，那么 view 就会把这个事件传递给自己的子 view：

```java
if (!canceled && !intercepted) {
    		...
            ...
            for (int i = childrenCount - 1; i >= 0; i--) {
                ...
                ...
                if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                    // Child wants to receive touch within its bounds.
                    mLastTouchDownTime = ev.getDownTime();
                    if (preorderedList != null) {
                        // childIndex points into presorted list, find original index
                        for (int j = 0; j < childrenCount; j++) {
                            if (children[childIndex] == mChildren[j]) {
                                mLastTouchDownIndex = j;
                                break;
                            }
                        }
                    } else {
                        mLastTouchDownIndex = childIndex;
                    }
                    mLastTouchDownX = ev.getX();
                    mLastTouchDownY = ev.getY();
                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                    alreadyDispatchedToNewTouchTarget = true;
                    break;
                }
                ...
                ...
            }
            
  			...
            ...
}
```

在这一个代码块里主要做的事情就是传递事件给子 view，首先对 view 的子 view 遍历，然后对每个子 view 调用了 dispatchTransformTouchEvent 方法，在这个方法里真正调用了每一个子 view 的 dispatchTouchEvent 方法，并获得子 view 的 dispatchTouchEvent 的返回值，如果有子 view 返回 true 即消费了这个事件，当前 view 就不会再继续传递给其他子 view 了，同时记录下这个信息。另外，如果这个事件被拦截了，就会通过 dispatchTransformedTouchEvent 调用到自己的父类的 dispatchTouchEvent，即在 View 中定义的 dispatchTouchEvent，在里面会调用到 view 的 onTouchEvent 方法，对事件进行处理，并返回是否消费。

上面所说的都是 ViewGroup 定义的 dispatchTouchEvent 方法，里面涉及到很大一部分的关于事件分发的逻辑，不过在 View 的 dispatchTouchEvent 里面，就没有这些了。

在一个 View 的 dispatchTouchEvent 方法里面，除了判断这个事件是否需要处理（ViewGroup 里也有这个判断），即产生当前事件的 activity、window 等是否还可用之类的，还使用 handleScrollBarDragging 方法判断这个事件是不是 scrollBar 的拖拽事件，如果是，消费掉，但是并不调用 onTouchEvent 方法，否则继续判断当前 view 是否设置了 onTouchListener，如果有，那么调用设置的 listener 的 onTouch 方法，查看返回值，如果为 true 则消费掉，并且不会再调用 自己的 onTouchEvent 方法，所以由此可以得到一个结论，**view的 setOnTouchListener 方法会优先于 view 的 onTouchEvent 方法调用**，只有当上述两个都表示自己不消费这个事件之后，才会调用 view 的 onTouchEvent 方法判断是否消费。最后，返回是否消费这个事件。

### onInterceptTouchEvent

这个方法只在 ViewGroup 里面有，因为一个纯粹的 view 并没有子 view，所以不需要考虑是否要分发给子 view，只需要直接调用自己的 onTouch 或者什么来判断是否消费即可。这个方法在 ViewGroup 里面默认返回的是 false，表示不拦截此次事件，将事件继续向下传递。这个方法主要是用来自定义一个 ViewGroup 的时候，重写这个方法来判断什么时候需要用户与当前 view 互动，什么时候用户是要与自己的子 view 互动，返回 true 即表示自己要处理这个事件，随之调用自己的 onTouchEvent，否则就会将这个事件传递给子 view，由子 view 进行分发。

### OnTouchEvent

在 ViewGroup 里面没有对 onTouchEvent 的重写，那么就可以直接看 View 里面的 onTouchEvent。在 View 的 onTouchEvent 中，首先判断了当前 view 是否是可点击的，如果是可点击的，就可以直接判断这个 view 会消费掉此次事件，相应的，可以直接在 xml 布局中使用 clickable 来指定一个 view 是否可点击，或者对于继承 view 的一些控件，如 Button，ImageButton，内置的就设置为可点击。另外当我们对一个 view 使用 setOnClickListener 设置了一个单机监听的时候，也会将这个 view 置为可点击，如：

```java
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}
```

如果一个 view 是可点击的，那么就可以直接在这个方法里面对点击事件进行判断了，比如监听单击、双击、长按什么的。

如果一个 view 有 TOOLTIP 这个 flag 也是会消费掉事件的，对 TOOLTIP 的描述为：

>   Indicates this view can display a tooltip on hover or long press.

表示 view 会对长按什么的作出反应，也表明了这个 view 是需要能点击的·。

如果这个 view 是不可点击的且不会对长按什么的作出响应，那就表示这个 view 不消费事件，返回 false。如果 view 消费了这个事件，那么这个事件就会在这里终止，否则的话，就会由 dispatchTouchEvent 方法回溯到上一层 view 的 dispatchTouchEvent 方法里面，然后再由上一层的 view 调用他的 onTouchEvent 方法判断是否消费这个事件，如果每一层的 onTouchEvent 里面都没有消费这个事件，那么这个事件就会一层一层的回到 activity 里面，调用 activity 的 onTouchEvent 方法。至此，一个 MotionEvent.ACTION_DOWN 的事件传送完毕了，它最初由 activity 中的 `getWindow().superDispatchTouchEvent()`方法一步一步送给各个 view 进行判断，最后还是会由自己将这个结果返回出去，而在这个过程中，各个 view 已经根据这个事件作出了自己该有的响应，用户与应用进行了一次友好的交流。

### 后续事件

不过事情并没有结束，这只是这一系列事件的第一个事件，而在这个事件中，还有不计可数的事件等待着处理，当然，确定了第一个事件的处理流程之后，后续事件都是根据这个事件的处理过程来处理的。因为在处理 MotionEvent.ACTION_DOWN 的时候，是谁消费了这个事件，这个事件都经过了哪些 view，这都是有记录的。比如第一次事件被 view A 消费了，那么当再有事件传输的时候，调用到 A 的 dispatchTouchEvent 方法，就不会再进行判断了，因为它消费了第一次事件，所以后面的事件都要由它来消费，所以就会直接调用这个 view 的 onTouchEvent 方法。不过如果下一个事件传递到 view A 的父 view B 的时候，在 B 的 onIntercrptTouchEvent 方法中返回的值是 true，那么这个事件就会被传递到 B 的 onTouchEvent 方法中，处理完之后就会从 B 向上传，不会再传到 A 了，那么这样的话就是说 A 的事件被 B 拦截了，在 A 里面就会传递一个 MotionEvent.ACTION_CANCEL，供 A 来处理后事，之后这一系列的后续事件的传递就会到 B 之后，就会往回走或者直接终止了。

## 关于MotionEvent.ACTION_CANCEL

ACTION_CANCEL 表示 当前手势操作被取消。

关于何时会响应 MotionEvent.ACTION_CANCEL 事件，一般的解释为 当用户的手指从当前控件移动到其他控件。比如说有一 ViewGroup a，它有一个子 View b， 手指最先点击在了 b 上面，然后慢慢移动手指，直到移除 b 的范围，并且触点还在 a 的范围上时，b 就会响应这个事件。

