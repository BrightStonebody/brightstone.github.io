---
title: ViewDragHelper的使用
date: 2020-02-19 10:36:30
tags:
- View
categories:
- Android
---

ViewDragHelper是一个V4包下的类, 可以帮助实现对子View的滑动拖放需求. 通常定义在自定义ViewGroup的内部

**参考**

[ViewDragHelper 的基本使用](https://www.jianshu.com/p/a9e0a98e4d42)

# 常用Api

**初始化**

```java
public static ViewDragHelper create(@NonNull ViewGroup forParent, @NonNull ViewDragHelper.Callback cb)
```

第一个参数是父布局, 第二个参数是自定义的监听回调

**拦截事件**

在自定义的ViewGroup中需要将点击事件交给ViewDragHelper来判断

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    //ViewDragHelper对事件进行拦截
    //注意, ACTION_DOWN事件不会被拦截
    //当clampViewPositionXxx方法没有修改left或top值时, 不会拦截
    //是的, shouldInterceptTouchEvent中会调用callback.clampViewPositionXxx方法
    return mViewDragHelper.shouldInterceptTouchEvent(ev);
}

@Override
public boolean onTouchEvent(MotionEvent event) {
    //将事件传递给ViewDragHelper进行处理
    mViewDragHelper.processTouchEvent(event);
    return true;
}
```

**处理computeScroll**

因为ViewDragHelper内部是通过Scroller来实现的, 所以要重写computeScroll方法(当Scroller处理滑动时, 会调用computeScroll方法)

```java
@Override
public void computeScroll() {
    super.computeScroll();
    if (mViewDragHelper != null && mViewDragHelper.continueSettling(true)){
        invalidate();
    }
}
```

**处理CallBack回调**

```java
private ViewDragHelper.Callback mCallback = new ViewDragHelper.Callback() {

    // tryCaptureView()我们可以在这个方法指定ViewGroup中的哪个子View可以被移动
    @Override
    public boolean tryCaptureView(View child, int pointerId) {
    }

    // clampViewPositionHorizontal() 判断水平方向上将要移动到的位置
    // left 将要移动到的left值
    // dx 表示速度
    @Override
    public int clampViewPositionHorizontal(View child, int left, int dx) {
    }

    // 同clampViewPositionHorizontal(), 换成了垂直方向而已
    @Override
    public int clampViewPositionVertical(View child, int top, int dy) {
    }

    // 手指拖拽后释放
    // releasedChild 拖拽的view
    // xvel, yvel拖拽动速度
    @Override
    public void onViewReleased(View releasedChild, float xvel, float yvel) {
    }
};
```

ViewDragHelper.Callback 是一个抽象类, 有很多可以重写的方法, 上面的是常用的方法

还有一些常用的关于边缘滑动相关的重写方法

在这里就不写了

## 例子

实现一个支持拖拽, 有粘性的ViewGroup

```java
public class DragLayout extends LinearLayout {

    private float mRaftValue = UIUtils.dip2Px(getContext(), 20); //回弹临界距离

    private float mRaftYVel = 1000.0f; // 回弹临界速度

    private ViewDragHelper mDragHelper;

    private OnDismissListener mOnDismissListener;

    private boolean mDragToggle = true;

    public DragLayout(@NonNull Context context) {
        this(context, null);
    }

    public DragLayout(@NonNull Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DragLayout(@NonNull Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    public void setOnDismissListener(OnDismissListener onDismissListener) {
        mOnDismissListener = onDismissListener;
    }

    private void init() {
        ViewDragHelper.Callback callback = new ViewDragHelper.Callback() {
            private int mCurrentTop;

            @Override
            public boolean tryCaptureView(@NonNull View child, int pointerId) {
                return mDragToggle;
            }


            @Override
            public int clampViewPositionVertical(@NonNull View child, int top, int dy) {
                // 禁止向上滑动
                if (top < 0) {
                    top = 0;
                }
                mCurrentTop = top;
                return top;
            }

            @Override
            public void onViewReleased(@NonNull View releasedChild, float xvel, float yvel) {
                super.onViewReleased(releasedChild, xvel, yvel);
                // 这里限定一个范围，在这个范围之内，可以进行回弹，大于这个范围，触发动画
                // 还需要限定一个速度，优化用户体验
                if (mCurrentTop <= mRaftValue && yvel <= mRaftYVel) {
                    mDragHelper.settleCapturedViewAt(0, 0);
                    invalidate();
                } else { // 从底部滑出
                    if (mOnDismissListener != null) {
                        mOnDismissListener.onDismiss();
                    }
                }
            }

            // 响应垂直滑动事件
            // 当子view会消费点击事件, 比如 clickable为true, 需要mDragHelper拦截点击事件
            // 而在mDragHelper.shouldInterceptTouchEvent(ev)方法中,
            // 当getViewVerticalDragRange和getViewHorizontalDragRange都返回0时, 不会拦截.
            // 所以需要重写并返回一个非0值
            @Override
            public int getViewVerticalDragRange(@NonNull View child) {
                return 1;
            }
        };

        mDragHelper = ViewDragHelper.create(this, callback);

    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {

        return mDragHelper.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        try {
            mDragHelper.processTouchEvent(event);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }

    @Override
    public void computeScroll() {
        if (mDragHelper != null && mDragHelper.continueSettling(true)) {
            invalidate();
        }
    }

    public interface OnDismissListener {
        void onDismiss();
    }
}
```

**注意** 在xml布局中, DragLayout必须只有一个直接子View, 这样才能实现对所有内容的拖拽