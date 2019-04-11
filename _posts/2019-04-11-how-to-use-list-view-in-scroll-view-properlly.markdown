---
layout:     post
title:      "如何在 ScrollView 里正确使用 ListView"
subtitle:   "避免知其然而不知其所以然"
date:       2019-04-11
author:     "ChengTao"
header-img: "img/android.png"
tags:
    - Android
---

> 昨天上线的 App 用户反馈了一个 Bug, 列表显示有 100 多项但是列表就是不能滚动, 一下搞得我也挺懵逼的, 就是一个简单的 ListView 而已, 怎么就滚动不了了呢? 原来是使用了一个自定义 ListView, 改成 ListView 后 Bug 也随之解决了.<br/><br/>
可是光解决 Bug 怎么可以呢? 我今年的目标可是成为合格的高级工程师, 必须彻底弄清楚原因才行. 于是花了一晚上看源码终于明白了其中的原因, 接着就有了这篇文章.

## 必备知识
1. 理解 View 的工作原理 (不理解请阅读 **<<Android 开发艺术探索>>** 第 4 章)

## 为什么项目中的自定义 ListView 无法滚动?
先看一下出现 Bug 的布局文件:
``` java
<RelativeLayout>
	<InScrollViewListView/>
</RelativeLayout>
```

再看看 InScrollViewListView 中的具体实现: 
``` java
public class InScrollViewListView extends ListView {

	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		// 强制修改 ListView 父控件的 MeasureSpec
		// size : Integer.MAX_VALUE >> 2 (View 的最大尺寸)
		// mode : MeasureSpec.AT_MOST
		int expandSpec = MeasureSpec.makeMeasureSpec(
			Integer.MAX_VALUE >> 2,
			MeasureSpec.AT_MOST
		);
		super.onMeasure(widthMeasureSpec, expandSpec);
	}

}
```
是不是非常熟悉的代码? 这么写一般是为了解决 ListView 在 ScollView 中只能显示一行 item 的问题, 这个问题我会在下面解释原因, 现在首要问题是为了弄明白为什么 InScrollViewListView 不能滚动的问题.

要解释 InScrollViewListView 为什么不能滚动就不得不看看 ListView 的 onMeasure 方法.

``` java
public class ListView extends AbsListView {

	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		
		......	

		final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
		int heightSize = MeasureSpec.getSize(heightMeasureSpec);
		// 此时 heightMode 和 heightSize 的值如下:
		// heightMode = MeasureSpec.AT_MOST
		// heightSize = Integer.MAX_VALUE >> 2 (View 的最大尺寸)
		
		......

		if (heightMode == MeasureSpec.AT_MOST) {
			// 计算 ListView 所有子控件的高度只和 (childrenHeight), 并和 heightSize 比较
			// 如果 childrenHeight > heightSize, heightSize = heightSize
			// 如果 childrenHeight < heightSize, heightSize = childrenHeight
			// 也就是说 ListView 高度不可能大于子控件的高度只和
			// 因为子控件只和不可能大于 Integer.MAX_VALUE >> 2, ListView 的高度为其子控件的高度之和
			heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
		}
		setMeasuredDimension(widthSize, heightSize);

		......
	}

}
```
由此可见 InScrollViewListView 的高度为子控件的高度之和, 既然 InScrollViewListView 的高度并没有小于子控件的高度之和, 又何谈滚动呢? **则项目中使用 InScrollViewListView 不能滚动的问题到这里就解释清楚了**.

## 为什么使用普通的 ListView 就解决了滚动问题?
要解释这个问题, 就必须看 RelativeLayout 的 onMeasure 方法, 因为 ListView 的 onMeausure 方法中的 heightMeasureSpec 是由 RelativeLayout 传递的.
``` java

public class RelativeLayout extends ViewGroup {

	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		
		......
	
		int myWidth = -1;
		int myHeight = -1;

		......

		final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
		final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
		final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
		final int heightSize = MeasureSpec.getSize(heightMeasureSpec);
		if (widthMode != MeasureSpec.UNSPECIFIED) {
			myWidth = widthSize;
		}
		if (heightMode != MeasureSpec.UNSPECIFIED) {
			myHeight = heightSize;
		}
		if (widthMode == MeasureSpec.EXACTLY) {
			width = myWidth;
		}
		if (heightMode == MeasureSpec.EXACTLY) {
			height = myHeight;
		}

		// 由于 RelativeLayout 传递过来的 MeasureSpec 的 mode 都是 mMeasureSpec.EXACTLY
		// 所以 myWidth 和 myHeight 都等于一个具体值, 这个由具体的设备来确定
		// myWidth 和 myHeigt 分别是 RelativeLayout 最终的宽和高
	
		......

		View[] views = mSortedHorizontalChildren;
		int count = views.length;

		// 横向测量子控件
		for (int i = 0; i < count; i++) {
			View child = views[i];
			if (child.getVisibility() != GONE) {
				LayoutParams params = (LayoutParams) child.getLayoutParams();
				int[] rules = params.getRules(layoutDirection);
				applyHorizontalSizeRules(params, myWidth, rules);
				// 这里去具体测量子控件, 主要工作就是计算子控件的 widthMeasureSpec 和 heightMeasureSpec, 再调用 View#measure 方法
				// 这里就 ListView 解释: 
				// 由于 ListView LayoutParam.height = LayoutParams.MATCH_PARENT
				// 所以传递到 ListView 的 heightMeasureSpec 由 MeasureSpec.EXACTLY 和 myHeight 组成
				measureChildHorizontal(child, params, myWidth, myHeight);
				
				......

			}
		}

		......

		// 纵向测量子控件
		for (int i = 0; i < count; i++) {
			final View child = views[i];
			if (child.getVisibility() != GONE) {
				final LayoutParams params = (LayoutParams) child.getLayoutParams();
				applyVerticalSizeRules(params, myHeight, child.getBaseline());
				// 这里和横向测量原理相同, 只不过计算具体 widthMeasureSpec 和 heightMeasureSpec 有所不同
				// 不过最终传递到 ListView 的 heightMeasureSpec 也是由 MeasureSpec.EXACTLY 和 myHeight 组成
				measureChild(child, params, myWidth, myHeight);

				.......

			}
		}
		
		......

	}

}
```
再来看看 ListView 的 onMeasure 方法:
``` java
public class ListView extends AbsListView {
	
	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		
		......	

		final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
		int heightSize = MeasureSpec.getSize(heightMeasureSpec);
		// 此时 heightMode 和 heightSize 的值如下:
		// heightMode = MeasureSpec.EXACTLY
		// heightSize = RelativeLayout 的高
		
		......

		setMeasuredDimension(widthSize, heightSize);

		......
	}

}
```
由此可见 ListView 高度为 RelativeLayout 的高度, 所以如果 ListView 的 item 项足够多, 以至于 item 的高度和大于 ListView 的高度, 则 ListView 便可以滑动.

## 一些思考
到这里我们已经对在 RelativeLayout 中 InScrollViewListView 为什么不可以滚动以及普通的 ListView 为什么可以滚动有了比较深的了解了, 但是别忘了 InScollViewListView 是为了解决 ListView 在 ScrollView 的高度为一条 Item 高度的问题, 接下来我们来一起研究一下以下 2 个问题:
1. 为什么 ListView 在 ScrollView 中高度只有一条 Item 的高度?
2. 如果更好的解决 ListView 在 ScrollView 中显示问题?

## 为什么 ListView 在 ScrollView 中高度只有一条 Item 的高度?
想解释这个问题, 我们必须看一下 ScrollView 的 onMeasure 方法:
``` java
public class ScrollView extends FrameLayout {

	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		// 调用父的方法, 即 FrameLayout#onMeasure 方法
		// 在下面的解释中, 可以得知 ListView 的高度为 0 或者 ListView 的第一个子控件的高度
		// 这个取决于 ListView 是否由子视图
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);

		// 这里判断 SrollView 设置了 mFillViewport
		// 可以在 xml 里设置 android:fillViewport
		// 也可以调用 ScrollView#setFillViewport 方法
		// 如果没有设置, 那么 ListView 的高度就只可能为 0 或者 ListView 的第一个子控件的高度
		if (!mFillViewport) {
			return;
		}

		final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
		if (heightMode == MeasureSpec.UNSPECIFIED) {
			return;
		}

		if (getChildCount() > 0) {
			final View child = getChildAt(0);
			final int widthPadding;
			final int heightPadding;
			final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
			final FrameLayout.LayoutParams lp = (LayoutParams) child.getLayoutParams();
			if (targetSdkVersion >= VERSION_CODES.M) {
				widthPadding = mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin;
				heightPadding = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin;
			} else {
				widthPadding = mPaddingLeft + mPaddingRight;
				heightPadding = mPaddingTop + mPaddingBottom;
			}
			
			final int desiredHeight = getMeasuredHeight() - heightPadding;
			// 如果设置了 mFillViewport, 并且 ListView 高度小于 ScrollView 的高度
			// 则 ListView 的高度就为 ScrollView 的高度
			// 因为 ScrollView 传递给 ListView 的 heightMode 为 MeasureSpec.EXACTLY
			if (child.getMeasuredHeight() < desiredHeight) {
				final int childWidthMeasureSpec = getChildMeasureSpec(
						widthMeasureSpec, widthPadding, lp.width);
				final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
						desiredHeight, MeasureSpec.EXACTLY);
				child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
			}
		}
	}

}
```
由于 ScrollView 继承 FrameLayout, 所以我们有必要看一下 FrameLayout 的 onMeasure 方法:
``` java
public class FrameLayout extends ViewGroup {

	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		int count = getChildCount();

		......

		for (int i = 0; i < count; i++) {
			final View child = getChildAt(i);
			if (mMeasureAllChildren || child.getVisibility() != GONE) {
				// 这里着重看这个方法, 因为这个方法 ScrollView 有重写
				measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
	
				......
				
			}
		}

		......

	}
}
```
接着我们来看一下 ScrollView 中的 measureChildWithMargins 方法的具体实现:
``` java
public class ScrollView extends FrameLayout {

	@Override
	protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
			int parentHeightMeasureSpec, int heightUsed) {
		final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

		final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
				mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
						+ widthUsed, lp.width);
		final int usedTotal = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin +
				heightUsed;
		final int childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(
				Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - usedTotal),
				MeasureSpec.UNSPECIFIED);
		// 由此可见 childHeightMeasureSpec 的 mode 为 MeasureSpec.UNSPECIFIED
		// child#measure 方法最终又会调用 ListView 的 onMeasure 方法
		child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
	}
}
```
最后我们再来重新看看 ListView 的 onMeasure 方法:
``` java
public class ListView extends AbsListView {

	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		
		......	

		final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
		int heightSize = MeasureSpec.getSize(heightMeasureSpec);
		// 此时 heightMode 为 MeasureSpec.UNSPECIFIED
		// 由 ScollView 的 measureChildWithMargins 方法传递过来
		
		int childWidth = 0;
		int childHeight = 0;
		int childState = 0;

		// 这里主要去计算 ListView 第一个子控件的宽度和高度
		mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
		if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED
				|| heightMode == MeasureSpec.UNSPECIFIED)) {
			final View child = obtainView(0, mIsScrap);

			// Lay out child directly against the parent measure spec so that
			// we can obtain exected minimum width and height.
			measureScrapChild(child, 0, widthMeasureSpec, heightSize);

			childWidth = child.getMeasuredWidth();
			childHeight = child.getMeasuredHeight();
			childState = combineMeasuredStates(childState, child.getMeasuredState());

			if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
					((LayoutParams) child.getLayoutParams()).viewType)) {
				mRecycler.addScrapView(child, 0);
			}
		}

		......
		
		// 这里可以看出, 当 heightMode == MeasureSpec.UNSPECIFIED 时
		// ListView 高度等于其一个子控件的高度加上 padding , 如果没有子控件就只剩 padding 了
		// 这里考虑一下没有 padding 的情况:
		// 1. 没有子控件 ListView 的高度为 0
		// 2. 有子控件 ListView 的高度为其第一个子视图的高度
		if (heightMode == MeasureSpec.UNSPECIFIED) {
			heightSize = mListPadding.top + mListPadding.bottom + childHeight +
					getVerticalFadingEdgeLength() * 2;
		}

		......

		setMeasuredDimension(widthSize, heightSize);

		......
	}

}
```
到这里我们已经通过源码了解了为什么 ListView 在 ScollView 中为什么显示一条 item 了, 其实这个说法并不准确, **准确说是当 ListView 有子控件的时候并且 ScrollView 没有设置 fillViewport, ListView 的高度为 ListView 第一个子控件的高度**.

## 如果更好的解决 ListView 在 ScrollView 中显示问题
从上面的源码分析中我们不难发现以下的几种方式:

1. 设置 ScollView 的 fillViewport 为 true
2. 重写 ListView 的 onMeasure 方法:
	1. 设置 heightMeasureSpec 为 MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2, MeasureSpec.AT_MOST), 这样 ListView 的高度就可以变成其子控件的高度之和
	2. 设置 heightMeasureSpec 为 MeasureSpec.makeMeasureSpec(MeasureSpec.getSize(heightMeasureSpec), MeasureSpec.AT_MOST), 这样 ListView 的高度就可以变成 ScrollView 的高度(如果 ListView 的子控件高度之和小于 ScrollView 的高度)

接下来我们来分析以下这些方法的优缺点:
- 方法 1 可以说就最佳的方式, 只用设置 ScrollView 的 fillViewport 就可以让 ListView 的高度变成 ScrollView 的高度
- 方法 2.1 应该是这里面最差的方式了, 虽然解决了 ListView 的显示的问题, 但是 ListView 的高度会变成其子控件的高度之和, 如果子控件非常多的的话, 由于 ListView 的复用机制不会起作用, 从而导致应用卡顿, 更则应用直接崩溃. 可能有一点大家会忽略, 为什么 ListView 这么设置以后还能滚动? 其实并不是 ListView 能滚动, 别忘了它的父控件是 ScrollView, ScollView 是可以滚动的, **这里并不是 ListView 的滚动, 是 ScrollView 的滚动**.
- 方法 2.2 是一个比较适中的方式, 能完美解决 ListView 的显示问题又不会破坏 ListView 的复用机制, 唯一的缺点就是必须新建一个类

## 小结
这次 ListView 和 ScrollView 分析应该算比较良心的了, 网上一搜 ListView 在 ScrollView 显示不正常就有一大堆博客安利 2.1 方法, 希望大家看到这篇文章后不要再误入歧途了, 以后的项目里就不要再使用 2.1 的方法了, 简直害人不浅.
