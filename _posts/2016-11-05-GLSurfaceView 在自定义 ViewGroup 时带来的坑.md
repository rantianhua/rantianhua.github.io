最近项目有个新的要求实现两个 View（分别记为 ViewA 和 ViewB ）的无限下拉效果，ViewA 和 ViewB 本身的内容完全不同，具体的要求有一下几点（最终最终的效果图在文章末尾）：
1. 不可以上拉切换两个View，如果是在手指拖动的过程中，View 可以进行上滑，但此时不能将 ViewA 和 ViewB 滑出屏幕。
2. 手指释放的时候，如果已经下拉了屏幕 1/3 的距离，则需要自动下拉切换 View。
3. 如果不满足要求2 ，则需要将 View 自动滑动到开始的位置，此时没有 View 切换。
4. 滑动时是两个 View 一起滑动，即 ViewA 被拉出了多少，ViewB 就要显示多少。
5. 通过代码调用也可以有下拉的动画效果（其实就是没有了手指拖拽的过程）
5. 上述操作可以无限操作（主要指下拉）

没有任何一个现成的控件能完全满足这些要求，所以还是自己手撸，自定义一个 ViewGroup 。主要的难点有两个：
+ 无限下拉
我需要控制两个 View ，让他们能不停地向下滑动。于是我在完成一次 View 切换之后（比如在显示 ViewA 显示时 切换 ViewB），需要将 ViewB 布局到 ViewA 的上面，这样当用户继续下拉时，可以再将 ViewB 拉出来，而且在每次切换完毕后都要进行这个操作：

```java

private void switchViewLayout(boolean changeView) {
    incrementVertical = 0;
    if(changeView) {
        //交换两个View的顺序，重新布局两个View
        childrens.add(0,childrens.remove(childrens.size()-1));
        requestLayout();
        showingView = childrens.get(childrens.size() - 1);
        if (pullDownSwitchListener != null) pullDownSwitchListener.onViewSelected(showingView);
    }
    if (pullDownSwitchListener != null) pullDownSwitchListener.onSwitchState(SwitchState.STATE_IDLE);
}

@Override
protected void onLayout(boolean c, int l, int t, int r, int b) {
    for (int i = 0; i < childrens.size(); i++) {
        View view = childrens.get(i);
        if (view.getVisibility() == GONE) continue;
        int top = (b - t) * (i - 1) / 2 + incrementVertical;
        int bottom = (b - t) * i / 2 + incrementVertical;
        view.layout(l,
                top,
                r,
                bottom);
    }
}

```
  注意在 onLayout 的时候，我始终将第二个 View布局到当前界面显示的位置 。
+ 拖动和自动下拉
让 View 动起来有三个方案可以选择：  
  1. scrollTo/scrollBy
  2. offsetTopAndBottom
  3. requestLayout    

一般会毫不犹豫选择方案一，但是我最终选择了第三个方案，而且也只有第三个方案能达到效果，因为其中一个 View 有个满屏的 GLSurfaceView。前两种原理是一样的，只是移动了 View 的内容，即通过移动 Canvas 将 View “滚动”起来，而 View 本身在 ViewGroup 中的位置没有发生改变。对于普通的 View 没有什么关系，但是 GLSurfaceView 就要区别对待了，因为它本身就是黑色的，如果只是内容滚动而位置不变，那么在滚动的过程中就会看到它的黑底（感觉就像衣服被扒了），这个黑底会把另一个被拉出的 View 给遮挡住（衣服被扒了不服气，非要耍流氓）。所以为了效果最终选择了效率不那么好的 requestLayout。  

至于自动下拉，一开始，我是自己模拟 Scroller 写了个 AutoLayouter 实现，之所以要写个 AutoLayouter，主要是因为我懒得将 `scroller.getCurrY()` 转换为 layout 需要的值，最重要的是，我想自己控制 `computeOffset` 的计算方式。后来还是发现自己想多了，直接用 ValueAnimator 也能完美满足我的要求：

```java
/**
 * @param start 开始的偏移量
 * @param end 最终的偏移量
 * @param changeView 是否切换了View
 * @param duration 动画时间
 */
private void startAnimator(int start, int end, final boolean changeView, int duration) {
    ValueAnimator valueAnimator = ValueAnimator.ofInt(start,end);
    valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator valueAnimator) {
            incrementVertical = (int) valueAnimator.getAnimatedValue();
            if(pullDownSwitchListener != null) {
                pullDownSwitchListener.onPullDownChanged(incrementVertical,onePageHeight);
            }
            requestLayout();
        }
    });
    valueAnimator.setDuration(duration);
    valueAnimator.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            switchViewLayout(changeView);
            isScrolling = false;
        }
    });
    isScrolling = true;
    valueAnimator.start();
}

```

做到这儿功能已经 OK 了，但还是有个小问题，就是在下拉 GLSurfaceView 时，和另一个 View 之间会有黑边闪烁，这个因为 GLSurafaceView 的绘制线程和 UI 线程没法同步的问题，目前也没有什么解决方案。所以在第二版，交互方式发生了改变，就是在下拉时，正在显示的 View 不动，只滚动隐藏的 View，就像拉窗帘一样，这样就规避了黑边的问题，于是另一个坑又来了。  

其实在现有自定义控件的基础上修改达到新效果很容易，只需要稍微修改下 `onLayout` 方法即可：

```java

@Override
protected void onLayout(boolean c, int l, int t, int r, int b) {
    for (int i = 0; i < childrens.size(); i++) {
        View view = childrens.get(i);
        if (view.getVisibility() == GONE) continue;
        //只移动之前没有显示的View，实现拉窗帘的效果
        boolean isShowingView = childrens.get(i) == showingView;
        int top = (b - t) * (i - 1) / 2 + (isShowingView ? 0 : incrementVertical);
        int bottom = (b - t) * i / 2 + (isShowingView ? 0 : incrementVertical);
        view.layout(l,
                top,
                r,
                bottom);
    }
}

```
修改后发现效果确实有了，但出现了一个很让人匪夷所思的问题，在通过代码调用切换的时候，没有 GLSurfaceView 的 View 是正常的，而有 GLSurfaceView 总是秒切，没有出现动画效果，不管切换时间设为多长。一开始我以为是什么操作导致该 View 的动画没有执行，可查看在 ValueAnimator 和 onLayout 两处的 log 信息，发现动画执行了（就是重新布局），两个 View 的代码表现完全一样，就是呈现的效果不同。

想了蛮久才搞明白是 View Z 轴顺序的问题，因为我先将 GLSurfaceView 添加进 ViewGroup，后添加普通 View ，所以普通 View 的 Z 轴顺序高于 GLSurfaceView，在我对 GLSurfaceView 进行不断 layout 的时候，它确实执行了，只不过是在普通 View 的下面执行了，所以在展示普通 View 的时候，看不到 GLSurfaceView 的拉窗帘效果，而之前的交互方式是两个都动，所以不关 Z 轴鸟事，WTF!

找到问题的原因就好解决了，只要我保证将要被拉出的 View 在 Z 轴的最上方就行了，于是 `bingToFront` 派上用场：

```java

@Override
public boolean onTouchEvent(MotionEvent event) {
    float curY = event.getY();
    switch (event.getAction()) {
        ...
        case MotionEvent.ACTION_MOVE:
            if (isScrolling) return false;
            final float interval = curY - preY;
            preY = curY;
            incrementVertical = (int) (curY - startY);
            if(incrementVertical < 0) {
                //不支持在原位置的基础上上拉
                incrementVertical = 0;
            }
            if (interval <= 0) {
                if (!isDragging) {
                    //不支持直接上滑操作,除非在已经 处于drag中
                    incrementVertical = 0;
                    return false;
                }
            }else {
                if (!isDragging) {
                    isDragging = true;
                    //保证看到动画效果
                    childrens.get(0).bringToFront();
                }
            }
            if(pullDownSwitchListener != null) {
                pullDownSwitchListener.onPullDownChanged(incrementVertical,onePageHeight);
            }
            requestLayout();
            break;
        ...
    }
    return true;
}

```

最后的效果图如下：

![最终效果图](http://upload-images.jianshu.io/upload_images/219854-adf2280645b09996.gif?imageMogr2/auto-orient/strip)

OK，希望这些坑能给大家带来帮助。
我已经把自定义的 ViewGroup 上传到 [Github](https://github.com/rantianhua/PullDownSwitchView.git) , 也有相应的例子，欢迎 Star。