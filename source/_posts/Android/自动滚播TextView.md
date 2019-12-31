---
title: 自动滚播TextView
date: 2019-12-31 14:49:08
tags:
- View
categories:
- Android
- View
---

要实现一个自动滚播的自定义View

```java
public class AutoSwitchView extends AppCompatTextView {

    private final int DEFAULT_IDLE_TIME = 3000;
    private final int DEFAULT_SWITCH_TIME = 1000;

    private int mIdleTime = DEFAULT_IDLE_TIME;
    private int mSwitchTime = DEFAULT_SWITCH_TIME;
    private Paint mPaint;
    private String mCurStr;
    private String mNextStr;
    private List<String> mContentList;
    private int mCurIndex = 0;
    private float mCurValue;
    private boolean mIsRunning = false;
    private ValueAnimator mAnimator;
    private Paint.FontMetrics mFontMetrics;
    private Runnable mRunnable;

    private int mWidth;
    private int mPaddingLeft;
    private int mPaddingRight;
    private int mPaddingTop;
    private int mPaddingBottom;
    private int mHeight;
    private float mTextBaseY;


    public AutoSwitchView(Context context) {
        super(context);
        init(context);
    }

    public AutoSwitchView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    public AutoSwitchView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context);
    }

    private void init(Context context) {
        mPaint = new Paint();
        mPaint.setTextAlign(Paint.Align.CENTER);

        mAnimator = ValueAnimator.ofFloat(0, 1);
        mAnimator.setStartDelay(mIdleTime);
        mAnimator.setDuration(mSwitchTime);
        mAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mCurValue = (float) animation.getAnimatedValue();
                // 必须要加这个判断, 不然会出现问题
                if (mCurValue < 1.0) {
                    invalidate();
                }
            }
        });
        mAnimator.addListener(new AnimatorListenerAdapter() {

            @Override
            public void onAnimationEnd(Animator animation, boolean isReverse) {
                if (!mIsRunning) {
                    return;
                }
                mCurIndex = (mCurIndex + 1) % mContentList.size();
                mCurStr = mContentList.get(mCurIndex);
                mNextStr = mContentList.get((mCurIndex + 1) % mContentList.size());
                if (mRunnable == null) {
                    mRunnable = new Runnable() {
                        @Override
                        public void run() {
                            mAnimator.start();
                        }
                    };
                }
                postDelayed(mRunnable, mIdleTime);
            }

            @Override
            public void onAnimationCancel(Animator animation) {
                mIsRunning = false;
            }
        });
    }

    public void start() {
        if (mContentList == null || mContentList.size() == 0)
            return;
        if (mIsRunning) {
            return;
        }
        if (mContentList.size() > 1) {
            mCurStr = mContentList.get(0);
            mNextStr = mContentList.get(1);
        } else {
            mCurStr = mContentList.get(0);
            mNextStr = mContentList.get(0);
        }
        mCurIndex = 0;
        mIsRunning = true;
        mAnimator.start();
        mAnimator.setStartDelay(mIdleTime);
    }

    public void stop() {
        if (mRunnable != null) {
            removeCallbacks(mRunnable);
        }
        mAnimator.cancel();
        mIsRunning = false;
        mCurIndex = 0;
    }

    public void setContentList(List<String> contentList) {
        if (contentList == null || contentList.size() == 0)
            return;
        mContentList = contentList;
        start();
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        mWidth = MeasureSpec.getSize(widthMeasureSpec);
        mPaddingLeft = getPaddingLeft();
        mPaddingRight = getPaddingRight();
        mPaddingTop = getPaddingTop();
        mPaddingBottom = getPaddingBottom();

        mPaint.setTextSize(getTextSize());
        mFontMetrics = mPaint.getFontMetrics();

        mTextBaseY = -mFontMetrics.top + mPaddingTop;
        mHeight = Math.round(mFontMetrics.bottom - mFontMetrics.top) + mPaddingTop + mPaddingBottom;

        setMeasuredDimension(mWidth, mHeight);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        float curStartX = mPaddingLeft + mPaint.measureText(mCurStr) / 2;
        float nextStartX = mPaddingLeft + mPaint.measureText(mNextStr) / 2;

        float baseY = 2 * mTextBaseY * (0.5f - mCurValue);
        if (baseY > 0) {
            canvas.drawText(mCurStr, curStartX, baseY, mPaint);
        } else {
            canvas.drawText(mNextStr, nextStartX, 2 * mTextBaseY + baseY, mPaint);
        }
    }
}

```