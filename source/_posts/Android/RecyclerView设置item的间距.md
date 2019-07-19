---
title: RecyclerView设置item的间距
date: 2019-07-09 17:29:55
tags:
- View
- RecyclerView
categories:
- Android
---

# RecyclerView设置item的间距

## 关于GridLayoutManager

当一个RecyclerView设置了一个GridLayoutManager(this,count)，并且count为4的时候，**实际上就是将屏幕均分为四份，每一份都是180px宽**（以720px为例，我们只考虑左右，暂不考虑上下，原理是相同的），如果不设置ItemDecoration，那么默认item由左开始布置。

## 自定义ItemDecoration

### getItemOffsets方法

设置ItemView的内嵌偏移长度

ItemView 外面会包裹着一个矩形（outRect）
内嵌偏移长度 是指：该矩形（outRect）与 ItemView的间隔
相当于为item添加了padding

**常用的getItemOffsets的计算**

```java
// 是item在GridLayoutManager中居中显示，并且item之间的间距，每列第一个item和每列最后一个item到recyclerview边界的距离都相等
public class EmojiItemDecoration extends RecyclerView.ItemDecoration {

    private int mNumColumn;
    private int mVerticalSpacing;
    private int mItemWidth;
    private int mItemHorizontalSpacing;
    private boolean mInitSpacing = false;

    public EmojiItemDecoration(int column, int itemWidth, int verticalSpacing) {
        mNumColumn = column;
        mItemWidth = itemWidth;
        mVerticalSpacing = verticalSpacing;
    }

    @Override public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {

        if (!mInitSpacing) {
            mInitSpacing = true;
            int parentWidth = parent.getWidth();

            mItemHorizontalSpacing = (parentWidth - parent.getPaddingLeft() - parent.getPaddingRight() - mItemWidth * mNumColumn) / (mNumColumn + 1);
        }
        int position = parent.getChildAdapterPosition(view);
        int column = position % mNumColumn;

        outRect.left = (mNumColumn - column) * mItemHorizontalSpacing / mNumColumn;
        outRect.right = (column + 1) * mItemHorizontalSpacing / mNumColumn;
        

        if (position < mNumColumn) {
            outRect.top = Dimensions.pxFromDp(12);
        }

        if (position >= mNumColumn) {
            outRect.top = mVerticalSpacing;
        }
    }
}
```

