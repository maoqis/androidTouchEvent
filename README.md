
[TOC] 

# 一、前言
## 为什么要做这个专题
### issue: DragLayout + SeekBar 滑动不灵敏,
拖动进度条时候经常 操作成了 拖拽DragLayout

- DragLayout是一个可以手机平面任意方向拖动的ViewGroup控件
- SeekBar就是我们说的进度条

### 在这之前对 touch事件的分发的了解

1. 从activity分发到根ViewGroup的dispatchTouchEvent 方法中。
2. ViewGroup 具有dispatchTouchEvent 、onInterceptTouchEvent、onTouchEvent三方法。
3. View 不需要（也没有）onInterceptTouchEvent方法。View的dispatchTouchEvent 事件默认调用onTouchEvent方法并返回onTouchEvent方法的返回值。
4. down事件进入ViewGroup 的dispatchTouchEvent 方法，dispatchTouchEvent 中遍历子view调用子view的dispatchTouchEvent 事件，如果处理了返回true，并且标记当前处理事件的子view，跳出循环遍历子view 返回true。ViewGroup 的子View都没有处理事件，调用ViewGroup的onTouchEvent方法。
5. 下一个move事件分发到到该ViewGroup时候，会将事件直接分发到Down事件时候标记的View中。
6. 如果ViewGroup的onTouchEvent方法没有处理，则表示该ViewGroup没有处理事件。此时事件向上返回（dispatchTouchEvent 返回false）。

### 半吊子的理论解决实际问题比较困难
于是决定重新研究下事件分发机制

# 二、流程分析
流程分析，源码分析，Demo验证


## 1. 流程图
### 事件分发图
- 图理解: 左边为一层递归调用，右边为递归返回
- 每一水平行，都属于 该组件的dispatchTouchEvent方法中的过程

![事件分发图](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-2.png)
- 什么是消费？不会继续往别的地方传了，事件终止。消费的过程=>向左retrun true，向上retrun
- Activity 返回true 或者 false 事件就被消费了（终止传递）, 总不能传给底部的stop状态下的Activity吧。



## 2. 源码分析
###  2.1 Activity.dispatchTouchEvent
- Activity.dispatchTouchEvent
```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

如果重写Activity的dispatchTouchEvent()方法，则会在分发事件前可处理触摸事件的相关逻辑. 另外此处getWindow()返回的是Activity的mWindow成员变量，该变量赋值过程是在Activity.attach()方法, 可知其类型为PhoneWindow.

- Activity.onTouchEvent
```
public boolean onTouchEvent(MotionEvent event) {
    //当窗口需要关闭时，消费掉当前event
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }

    return false;
}
```
- PhoneWindow.superDispatchTouchEvent
```
public boolean superDispatchTouchEvent(KeyEvent event) {
    return mDecor.superDispatcTouchEvent(event); 
}
```

PhoneWindow的最顶View是DecorView，再交由DecorView处理。而DecorView的父类的父类是ViewGroup,接着调用 ViewGroup.dispatchTouchEvent()方法。为了精简篇幅，有些中间函数调用不涉及关键逻辑，可能会直接跳过。

### 2.2  ViewGroup.dispatchTouchEvent

#### 拦截
![拦截](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-3.png)
- 时机: down事件 或 有mFristTouchTarget时候才需要 判断是否拦截，其他情况直接拦截
- mFristTouchTarget 标记消耗了事件的子View 
- disallowIntercepter 不准许拦截

####  Down事件分发
- 不拦截时候执行, targetView设置、 处理down事件分发



![down事件分发](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-4.png)

- 处理down 事件的分发，添加Down事件的TouchTarget，并做个标记处理过分发

![down事件分发](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-5.png)

#### 其他事件分发
通过TouchTarget 进行直接分发


![其他事件分发](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-6.png)


####  调用onTouchEvent
down 事件后如果没有 mFirstTouchTarget，那targetView一直为空

![调用onTouchEvent](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-7.png)


![调用onTouchEvent](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-8.png)

## 3. Demo 测试


### 3.1 dispatchTouchEvent

- dispatchTouchEvent Down事件返回false，就收不到接下来的其他事件。


![dispatchTouchEvent Down](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-9.png)

```

事件要么 被兄弟View消耗后标记成mFirstTouchTarget，下次直接分发

要么父View 自己处理 ，它的mFirstTouchTarget = null
->不调用 onInterceptTouchEvent，intercepted = true 
->跳过 touch targets->直接处理

要么父View dispatchTouchEvent 返回false ，向上类推
```
### 3.2 onInterceptTouchEvent

一旦onIntercepTouchEvent返回true

#### a.  Down事件: <br>


不分发当前事件，ViewGroup onTouchEvent处理事件 

```
            
            //拦击后返回 intercepted = true
            
            
            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
                //down 事件的分发
            }
```
down事件分发过程后 设置的 mFirstTouchTarget，所以mFirstTouchTarget为null
```
// Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            }
```
#### b. move事件 onInterceptTouchEvent返回true<br>

- 前提:先不拦截

之前收到的事件的 dispatchTouchEvent 返回true。父ViewGroup不拦截；
     自己的 onTouchEvent 返回 true。
     


![onInterceptTouchEvent Down](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-10.png)
mFirstTouchTarget 不为null

- 设置拦截

![onInterceptTouchEvent 设置拦截 Down](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-11.png)

当前Move事件变成 Cancel 事件，分发给子targetView；之后事件不分发子View

源码分析: 
经过前提后，mFirstTouchTarget不为空

![onInterceptTouchEvent 设置拦截 Down](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-12.png)

#### c. 之后事件不分发子View
mFirstTouchTarget = null
```
mFirstTouchTarget = null //跳过了 拦截处理代码
intercept变量= true ||  mFirstTouchTarget = null // 跳过的 down事件分发
mFirstTouchTarget = null // 调用onTouchEvent

```
### 3.3 onTouchEvent
#### Down时候 返回false
等同 Down时候 dispathOnEvent 返回false，收不到后续事件
- =>如果子View 需要处理事件，Down事件一定返回true

#### Down时 返回true
- 如果父View 不拦截事件，则一直收到事件 

#### Down 事件返回true，如果后续事件都返回true
- 比较普遍，eg: SeekBar

#### Down 事件返回true，如果后续事件全部或者部分返回false
- 也一直收到事件,因mFirstTouchTarget不为空
- dispathTouchEvent 返回 false
- 可能Activity onTouchEvent 能收到事件
- 具体应用场景不知晓










# 三、问题
### 一般什么时候消费事件
SeekBar TouchEvent  一直返回true的消费事件
DragLayout 的TouchEvent 返回true的消费事件
dispatchTouchEvent 由TouchEvent 引起的消费事件，当然也能自己从写返回true消耗事件


### onClick 、onLongClick  、- OnTouchListener 
- 在 View 的onTouchEvent中触发，onClick 、onLongClick 在收到up事件时候触发
- OnTouchListener 在View 的dispatchOnEvent 中触发。如果设置了OnTouchListener dispatchOnEvent返回true，不再走onTouchEvent


```
    伪代码
    dispatchTouchEvent() {
        if (onTouch) {
            return true;
        }
        
        .....
        
        onTouchEvent {
            switch
                case up
                   执行 onClick 等事件
                    
        }
        
    }

```

### 点击坐标的影响
Demo 测试 2.1 中 点击了祖父ViewGroup范围区域，
事件分发不经过 parent和child

```
//如果view不可见，或者触摸的坐标点不在view的范围内，则跳过本次循环
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }
```
# 四、场景
## 1. DragLayout 和 SeekBar 结合
- DragLayout 是 SeekBar 上寻ViewGroup
- 在SeekBar上方 水平拖动时候 ，SeekBar进行拖动
- 竖直滑动时候，DragLayout 进行滑动
- SeekBar进行拖动时候，DragLayout 不能滑动
- DragLayout滑动时候， SeekBar不进行拖动

### 1.1 事件一开始 分发到DragLayout 和 SeekBar上
- SeekBar 的onTouchEvent 返回true


![onInterceptTouchEvent 设置拦截 Down](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-13.png)

- DragLayout 开始不能拦截事件，尤其是Down事件
### 1.2 怎么判断是 SeekBar 或 DragLayout 拖动
SeekBar 是在onTouchEvent中判断，DragLayout是在 onInterceptTouchEvent中判断
判断方法都是收到move事件时候， 计算距离move和down事件时候 事件的点坐标的距离 是否到达可拖动的界限

### 1.3 子View 可以通过 requestDisallowInterceptTouchEvent 禁用父VIew 拦截事件
SeekBar 中具体实现 拖动超过界限 执行下面方法


![SeekBar](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-14.png)



![SeekBar](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-15.png)

demo 演示：Parent 相当于 SeekBar


![SeekBar ](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-16.png)
注意: DragLayout之前只是在onInterceptTouchEvent 中收到过 Down事件，不用发送Cancel 取消事件；DragHelper 中下次收到Down事件时候自然会再清空之前未清空的事件



### 1.4 父ViewGroup 可以通过 onInterceptTouchEvent 返回true 自己处理事件

![父ViewGroup ](https://raw.githubusercontent.com/maoqis/androidTouchEvent/master/image-17.png)



#### DragHelper.shouldInterceptTouchEvent

- move事件中处理

shouldInterceptTouchEvent 返回true ，需要 mDragState == STATE_DRAGGING

```
    public boolean shouldInterceptTouchEvent(@NonNull MotionEvent ev) {
        ..省略
        
        //move时候处理, tryCaptureViewForDrag中 设置 mDragState =STATE_DRAGGING
        if (pastSlop && tryCaptureViewForDrag(toCapture,pointerId)) {
            break;
        }
        
        return mDragState == STATE_DRAGGING;// 如何返回true
    }
```


就上面代码中需要 pastSlop == true，才能执行&&之后tryCaptureViewForDrag 方法。

看下之前代码 pastSlop 如何取值
```
//move事件

final boolean pastSlop = toCapture != null && checkTouchSlop(toCapture, dx, dy);

```
需要下面方法返回true，滑动大于阀值
```
    private boolean checkTouchSlop(View child, float dx, float dy) {
        if (child == null) {
            return false;
        }
        final boolean checkHorizontal = mCallback.getViewHorizontalDragRange(child) > 0;
        final boolean checkVertical = mCallback.getViewVerticalDragRange(child) > 0;

        if (checkHorizontal && checkVertical) {
            return dx * dx + dy * dy > mTouchSlop * mTouchSlop;
        } else if (checkHorizontal) {
            return Math.abs(dx) > mTouchSlop;
        } else if (checkVertical) {
            return Math.abs(dy) > mTouchSlop;
        }
        return false;
    }
```



### 1.5 水平滑动时候让 SeekBar 拖动判断先执行
- 让DragLayout 计算拖拽距离（move点到down点距离）是否超过 阀值时候，只计算竖直方向间距
- 换句话就是 DragLayout不处理 水平方向拖拽
```
让checkTouchSlop方法中 checkHorizontal = false
```
需要 mCallback.getViewVerticalDragRange(child) <=0

### 1.6 梳理流程
用户滑动时候 一般水平和竖直方向都有 移动距离

#### 在SeekBar 区域滑动

- 用户没有拖动
```
正常分析，相当于Demo中只开 SeekBar TouchEvent = true 开关

DragLayout onInterceptTouchEvent 返回false
SeekBar  TouchEvent 返回 true 

```
DragLayout onInterceptTouchEvent 收到过Down、Move、Up 等事件

- 用户水平拖动时候
```
Down事件分发到 DragLayout 和 SeekBar
Move事件
    开始的Move事件可能计算距离不够拖动阀值,
    SeekBar TouchEvent 
        判断开始拖动了=> 请求父View不要拦截事件
此时你再怎么竖直滑动 ，因父View 不走InterceptTouchEvent方法，也不会竖直方向有滑动
    SeekBar TouchEvent
```

DragLayout onInterceptTouchEvent 收到过Down、部分Move 等事件

- 用户竖直拖动时候

```
Down事件分发到 DragLayout 和 SeekBar

开始的Move事件可能计算距离不够拖动阀值，

之后的Move或Up事件
    DragLayout  onInterceptTouchEvent 判断开始拖动了=> 父View处理事件
        变发 Cancel事件 到 SeekBar，TargetView 清空
        DragLayout TouchEvent 该m事件
此时你再怎么水平滑动 ，因父自己处理事件，也不会竖直方向有滑动
        
```

DragLayout onInterceptTouchEvent 收到过Down、部分Move 等事件

DragLayout TouchEvent 收到剩下的Move 和 Up 等事件

#### 在DragLayout 中 非 SeekBar区域

```
正常分析，相当于Demo中只开 SeekBar TouchEvent = true 开关

DragLayout onInterceptTouchEvent 返回false
子View  TouchEvent  都返回 false
DragLayout TouchEvent 处理

```
DragLayout onInterceptTouchEvent Down 和开始部分事件，可能收到所有事件（不滑动情况）

DragLayout TouchEvent 收到Down、Move 和 Up 等事件


## 2. SwipeBackLayout
https://github.com/gongwen/SwipeBackLayout/blob/master/library/src/main/java/com/gw/swipeback/SwipeBackLayout.java

类似 DragLayout 使用 DragHelper

## 3. ViewPager 
雷同 DragLayout

https://www.jianshu.com/p/b8fe093a9d4b

手势 Gesture , touch



# 总结
从事件分发流程图，源码讲解分发过程，使用Demo测试各种分发情况。
结合DragLayout + SeekBar 实战情况讲解了ViewGroup 和View 之前滑动事件如何处理。

## 实战点:

1. ViewGroup要自己处理事件, 不重写ViewGroup 的dispatchTouchEvent方法
2. ViewGroup想要处理事件 在 onInterceptEvent中返回true。一旦返回true，如果子View之前收到过事件 则该发Cancel事件，之后收不到事件；如果子View之前没有收到过，则继续收不到事件。
3. 子View处理事件，此时父View可以拦截，为了不让父View拦截，通过 requestDisallowInterceptTouchEvent，保证子View继续处理事件。


## 文献
- https://www.jianshu.com/p/e99b5e8bd67b
- https://www.jianshu.com/p/42dfd6a27c61
- http://gityuan.com/2015/09/19/android-touch/
- 感谢佟磊哥提供的事件分发Demo
