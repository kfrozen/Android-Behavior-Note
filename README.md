## CoordinateLayout和Behavior的那些事 ##

本质上来说，CoordinateLayout就是一个FrameLayout，所以在其子View没有设置任何Behavior的情况下，它做的事情就跟FrameLayout是一样的。这里主要讨论Behavior两个重要的功能，measure/layout和scrolling event的处理。

**1. Measure/Layout**

	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

		......

		final int childCount = mDependencySortedChildren.size();
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
			......
			final Behavior b = lp.getBehavior();

			/***只有当该child没有设置behavior或者其behavior并没有处理onMeasureChild方法时才会调用自身的onMeasureChild去处理该child
            if (b == null || !b.onMeasureChild(this, child, childWidthMeasureSpec, keylineWidthUsed,
                    childHeightMeasureSpec, 0)) {
                onMeasureChild(child, childWidthMeasureSpec, keylineWidthUsed,
                        childHeightMeasureSpec, 0);
            }
			......
		}

		......

	}

	@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior behavior = lp.getBehavior();

			/***只有当该child没有设置behavior或者其behavior并没有处理onLayoutChild方法时才会调用自身的onLayoutChild去处理该child
            if (behavior == null || !behavior.onLayoutChild(this, child, layoutDirection)) {
                onLayoutChild(child, layoutDirection);
            }
        }
    }

	这是CoordinateLayout的onMeasure(...)和onLayout(...)方法，可以看出此处会遍历所有的子View并且委托它们的behavior去进行measure/layout

下面这个类实现了对child views布局的功能，可以参考

	/**
 	* The {@link Behavior} for a scrolling view that is positioned vertically below another view.
 	* See {@link HeaderBehavior}.
 	*/
	abstract class HeaderScrollingViewBehavior extends ViewOffsetBehavior<View>

**2. Scrolling Event**

CoordinateLayout实现了NestedScrollingParent接口，但其对于嵌套滚动事件的实现也是回调子View的behavior中的相关方法实现的，如下

	@Override
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed) {
        final int childCount = getChildCount();
        boolean accepted = false;

        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                viewBehavior.onNestedScroll(this, view, target, dxConsumed, dyConsumed,
                        dxUnconsumed, dyUnconsumed);
                accepted = true;
            }
        }

        if (accepted) {
            onChildViewsChanged(EVENT_NESTED_SCROLL);
        }
    }

当用户做出scroll/fling的动作时，如果需要对CoordinateLayout中的某个子View做相应的处理，可以在behavior的nested scrolling相关的方法中去做。


----------


这里用**AppBarLayout**来举例，AppBarLayout中包含有两个Behavior的实例：**a.** public static class **Behavior** extends HeaderBehavior<AppBarLayout>; **b.** public static class **ScrollingViewBehavior** extends HeaderScrollingViewBehavior

其中，Behavior是给AppBarLayout自己用的，它主要响应并处理了来自CoordinateLayout的Nested Scrolling事件，并且会带动AppBarLayout本身进行平滑的位移。

而ScrollingViewBehavior，顾名思义是给AppBarLayout的Sibling View(即平级的View)使用的，通过源码我们可以看到这个Behavior其实没有做也没有响应任何与Nested Scrolling相关的事情和方法。它的核心代码都在onMeasureChild(...)和onLayoutChild(...)中，这两个方法均是会被CoordinateLayout调用。这两个方法确定了这个View将如何在CoordinateLayout中被摆放(由于CoordinateLayout本质上可以被理解为一个FrameLayout，如果其子View没有设置类似的Behavior，则所有子View将会被重叠摆放)。再看下面这两个方法：

		@Override
        public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
            // We depend on any AppBarLayouts
            return dependency instanceof AppBarLayout;
        }

        @Override
        public boolean onDependentViewChanged(CoordinatorLayout parent, View child,
                View dependency) {
            offsetChildAsNeeded(parent, child, dependency);
            return false;
        }

layoutDependsOn(...)决定了这个View所依赖的Sibling View，这个例子中明显是AppBarLayout，所以设置了这个Behavior的View在CoordinateLayout中会被摆放在AppBarLayout的下方。

onDependentViewChanged(...)方法中定义了当dependentView发生变化时，该View做出如何的反应。在AppBarLayout中，这个View会跟着AppBarLayout进行移动，这样就实现了整体页面的纵向平移，但其实ScrollingViewBehavior并没有处理和响应Scrolling事件，它只是跟着设定好的依赖View进行动作。
