---
layout: post
category: Android
title: iOS版Teambition菜单交互的Android实现
tags: Teambition菜单，Android
keywords: Teambition菜单, 自定义View
excerpt: Teambition菜单Android实现
redirect_from:
  - /2017/03/Teambition首页菜单剖析/
---

话说在春节的时候闲着没事搞了这个博客，一晃过去快两个月了也没更新过，再一次被自己折服，哈哈哈哈哈哈。OK，今天更第一篇，就写点关于Android UI的小东西。

## 背景

这篇软文的知识背景，说破天就是一个Android的自定义View，我的文章并不是教程类的，所以我默认大家是理解自定义View的基本流程和原理。

不知大家了不了解**Teambition**这个产品，反正我之前是没听过，但是因为需要，要搞它这个交互，哈哈哈。不过呢，体验起来感觉还是不错的，先来看下效果吧。

![](/assets/postimage/effect.gif){:width="300"}

这个效果是iPhone版本的，Android版并没有这个交互，乍一看这个，和经典的卫星菜单是有相似之处的，那说起来就比较好实现了，然而并不完全一样，所以在Android上实现还是要靠自己去分析一下。

## 解析

我所解析的是我通过体验这个交互分析出的原理，并不一定是他实现的真正原理。首先，这个菜单分为两个部分 —— 背景和菜单内容。这两个元素在这个过程中都是变换的动画效果，所以这个过程中是有个动画作为衔接，而这个动画也是这个交互效果是否够炫酷的关键。


**一、** 首先看背景，还是两部分，渐变的半透明白色背景和蓝色圆弧背景。白色半透明这个效果这个对iOS来说实现可能很简单，可能直接拍个毛玻璃比你Android折腾半天效果还好，对Android来说直接通过 `setAlpha()` 这个方法并不能达到这个效果，不信邪的可以试试。但是Android是不输给iOS的，曲线救国也是有办法的，View类还提供了一个方法`setBackgroundColor(int color)`，这个方法参数是个颜色的值，这样就可以通过动态传入argb颜色值达到这个效果。


下面这个弧形背景相对复杂些，但以初中的数学能力，完全可以理解了，为了更直观，先上张图


![](/assets/postimage/3-14-background.png)


图中，蓝色的部分是我们想要绘制的背景，水平方向上是对称的。既然是弧，我们很容易想到圆，所以关键是找到圆心，求出半径。那么半径怎么求？相信看了这个图大家应该一目了然了。UE标尺寸是通过圆弧最高点距屏幕底部的距离 **H**（实线部分）和圆弧和屏幕左右的交点距离底部的距离 **h** 来控制圆弧的程度，那么我们通过这两个距离也正好足以算出半径 **R** 的大小，即三角形 **OPQ** 的勾股定理，不再赘述，直接给出推导出的公式

> R = a/2 + w²/ 8*a   （a=H-h）

求出半径 **R** 就可以找到圆心的坐标，有了圆心和半径，可以画圆了，但是，观察效果图可以发现，在菜单打开和关闭过程中，背景的开始消失都是收敛在底部的按钮上，所以说这个圆心在这个过程中并不是固定的，是动态转移的，移动的区间就是以上所求出的圆心坐标和按钮坐标之间，Y轴的变化。

**二、** OK，背景搞定了，那么菜单的item是什么原理呢？效果图可以看出，菜单的放置也是跟随着背景弧的弧度，那么我们就能确定，这两个圆心是同一个，只不过半径有差别，显然，菜单的半径 **r** 小于背景弧的半径 **R** 。

`r = R-菜单离边界的距离`

现在菜单的半径有了，下一步是如何利用这个半径计算出每一个item的位置坐标，进而去layout每一个item，再看张老图：

![](/assets/postimage/3-14-itemlayout.png)

图中我标出两个角度，∠a和∠b。为了让菜单放置在屏幕内不超出且分布合理，∠b就是我们菜单项的layout范围。有人可能会有疑问，为什么不是∠SOP之间也不是∠TOU之间，非要弄个中点，造出这么个奇怪的角度？其实那两个角度范围我也试了，在layout的时候布局效果不佳，有兴趣的可以试试。而∠b范围内菜单布局是相对完美的。而且这个角度并不难求：

> ∠b = arctan( (w/2) / (offsetY+h/2) ) * 2    
(w=屏幕宽度，offsetY=圆心与屏幕底端的距离)

这样的话∠a也就有了：

> ∠a = (π - ∠b)/2

求出这两个角度，再加上之前算出的半径，就可以算出每个item的坐标，直接给公式


> l = w / 2 - r \* cos(∠b / count \* i + ∠a) - itemWidth / 2   
t = h - (r \* sin(∠b / count \* i + ∠a) - offsetY)   
(count = item数量 ，i是item的索引，中间按钮的索引是0，菜单项i从1开始)

* *注意，计算 l 的时候减了一个itemWidth/2，是因为计算位置是以每个item的中心为标准，实际求的参数l是item的左边界，如果不减去 itemWidth/2 那么每个item的位置都会偏右，可以试一下。*

再次强调，理解以上这些公式需要对Android的自定义控件和屏幕坐标系有一定理解。

**三、** 前两部分搞定基本搞定这个菜单的原形，但还需要一个关键的东西——**动画**。先来分析一下这个过程中，哪些元素需要动画。有菜单item，圆弧背景绘制，白色背景透明度，圆弧圆心位置。那么，我们把动画实现也分为两部分，item动画和背景动画，分别用tween动画和属性动画去实现。

item动画只是上下的位移和出现消失，用tween动画比较好，显然是`TranslateAnimation`和`AlphaAnimation`组合的`AnimationSet`,实现比较简单不在赘述。

背景动画显然是属性动画，但我们需要用一个动画去控制三个东西，而每个元素的变化范围是不同的，所以采用单位区间去统一管理，即用`ValueAnimator`设置单位区间`[0,1]`，那么这个区间就可以映射到任意区间了，完美。

## 实现

我相信看了以上解析，大家对这个控件的实现已经基本有了思路，前边说的比较多就是为了能达到这样的效果。要相信，程序员的大多数时间绝不是在写代码，而是在思考！哈哈哈，师父教的。
接下来主要是代码实现，可看可不看。

实现主要步骤:
1. 定义属性并在构造方法中解析
2. 实现`onMeasure` 方法确定大小
3. 实现`onLayout`方法确定每个元素位置
4. 实现`onDraw`绘制背景
5. 为各个元素添加动画
6. 处理Touch事件
7. 提供对外能力

自定义控件，首先抽象出自定义属性：
```
<declare-styleable name="BottomMenu">
    <attr name="menu_marginBottom" format="dimension" />
    <attr name="menu_backgroundArcHeghit" format="dimension" />
    <attr name="menu_backgroundHeight" format="dimension" />
    <attr name="menu_item_marginEdge" format="dimension"/>
    <attr name="menu_backgroundColor" format="color" />
    <attr name="menu_animDuration" format="integer" />
</declare-styleable>
```


字面意思也基本可以理解，其中有一个`menu_marginBottom`,是按钮与底部的距离，这样便于调整位置，这个变量的存在也会稍微影响前边的公式形式，但原理不变。`menu_backgroundArcHeghit`对应**H**，`menu_backgroundHeight`对应小**h**，别的没啥说的。

再看变量：
```java
    /**
     * 默认值
     */
    private static final int DEFAULT_ANMI_DURATION = 300;
    private static final int DEFAULT_ARC_HEIGHT = 180;
    private static final int DEFAULT_HEGIHT = 150;


    /**
     * 菜单弹出半径
     */
    private int mRadius;
    /**
     * 菜单距离底部距离  便于调整位置
     */
    private int mMarginBottom;
    /**
     * 主菜单按钮
     */
    private View mButton;


    /**
     * 背景弧最终半径
     */
    private int mBackgroundRadius;
    /**
     * 绘制过程中背景弧的动态半径
     */
    private int mDrawingBackgroundRadius;
    /**
     * 背景高度
     */
    private int mBackgrondheight;
    /**
     * 背景弧高（最高点）
     */
    private int mBackgroudArcHeight;

    /**
     * 菜单元素距边缘的距离
     */
    private int mItemMarginEdge;

    /**
     * 菜单放置的角度区间
     */
    private double mItemAngleSection;
    /**
     * 背景色
     */
    @ColorInt
    private int mBackgroudColor;
    // 画背景圆心坐标
    private int mDrawX;
    private int mDrawY;

    /**
     * 圆心与屏幕底边距离
     */
    private int mPointOffetScreenY;

    private int mStartDrawingY;
    private int mDrawingY;
    private int mDrawYOffset;

    private Paint mPaint;

    // 对外提供的接口
    private OnMenuItemClickListener mOnMenuItemClickListener;
    private OnAnimationRunningListener mOnAnimationRunningListener;

    // 状态
    private Status mStatus = Status.CLOSE;

    @IntRange(from = 10, to = 2000)
    private int mAnimDuration;
    private boolean isAnimRunning = false;
```

解析代码就不贴了，下面看测量。为了让控件适应性更强，不一定非要在全屏下使用，需要写一下这个`onMeasure`方法：

```java
   @Override
   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

       int widthMode = MeasureSpec.getMode(widthMeasureSpec);
       int widthSize = MeasureSpec.getSize(widthMeasureSpec);

       int heightMode = MeasureSpec.getMode(heightMeasureSpec);
       int heightSize = MeasureSpec.getSize(heightMeasureSpec);

       for (int i = 0; i < getChildCount(); i++) {
           View v = getChildAt(i);
           int widthSpec = 0;
           int heightSpec = 0;
           LayoutParams params = v.getLayoutParams();
           if (params.width > 0) {
               widthSpec = MeasureSpec.makeMeasureSpec(params.width, MeasureSpec.EXACTLY);
           } else if (params.width == -1) {
               widthSpec = MeasureSpec.makeMeasureSpec(widthSize, MeasureSpec.EXACTLY);
           } else if (params.width == -2) {
               widthSpec = MeasureSpec.makeMeasureSpec(widthSize, MeasureSpec.AT_MOST);
           }

           if (params.height > 0) {
               heightSpec = MeasureSpec.makeMeasureSpec(params.height, MeasureSpec.EXACTLY);
           } else if (params.height == -1) {
               heightSpec = MeasureSpec.makeMeasureSpec(heightSize, MeasureSpec.EXACTLY);
           } else if (params.height == -2) {
               heightSpec = MeasureSpec.makeMeasureSpec(heightSize, MeasureSpec.AT_MOST);
           }
           v.measure(widthSpec, heightSpec);
       }

       if (widthMode != MeasureSpec.EXACTLY) {
           widthSize = Math.min(widthSize, getChildAt(1).getMeasuredWidth() * (getChildCount() - 1));
       }
       if (heightMode != MeasureSpec.EXACTLY) {
           heightSize = mBackgroudArcHeight;
       }
       setMeasuredDimension(widthSize, heightSize);

   }
   ```

其实主要是处理子View的测量和`wrap_content`的情况.下面是`onLayout`方法：

```java
  @Override
  protected void onLayout(boolean changed, int l, int t, int r, int b) {
      if (getChildCount() < 2) {
          throw new IllegalStateException("At least one menu item");
      }
      layoutCenterButton();
      layoutChildItems();
  }

  private void layoutCenterButton() {

      mButton = getChildAt(0);
      int bWidth = mButton.getMeasuredWidth();
      int bHeight = mButton.getMeasuredHeight();
      int bX = getMeasuredWidth() / 2 - bWidth / 2;
      int bY = getMeasuredHeight() - bHeight - mMarginBottom;
      mButton.layout(bX, bY, bX + bWidth, bY + bHeight);
      mButton.setOnClickListener(this);

      mDrawX = getMeasuredWidth() / 2;
      int h = mBackgroudArcHeight - mBackgrondheight;
      mBackgroundRadius = (int) (h / 2 + Math.pow(getMeasuredWidth(), 2) / (8 * h));
      mDrawY = getMeasuredHeight() + mBackgroundRadius - mBackgroudArcHeight;

      mStartDrawingY = getMeasuredHeight() - mMarginBottom - bHeight;
      mDrawYOffset = mDrawY - mStartDrawingY;

      mRadius = mBackgroundRadius - mItemMarginEdge - mMarginBottom;
      mPointOffetScreenY = mDrawY - getMeasuredHeight();
      mItemAngleSection = Math.atan((double) mDrawX / (mPointOffetScreenY + mBackgrondheight / 2)) * 2;


  }

  private void layoutChildItems() {
      int count = getChildCount();
      double baseAngle = (Math.PI - mItemAngleSection) / 2;
      for (int i = 1; i < count; i++) {
          View child = getChildAt(i);
          int cWidth = child.getMeasuredWidth();
          int cHeight = child.getMeasuredHeight();

          int l = (int) (getMeasuredWidth() / 2 - mRadius * Math.cos(mItemAngleSection / count * i + baseAngle) - cWidth / 2);
          int t = (int) (getMeasuredHeight() - (mRadius * Math.sin(mItemAngleSection / count * i + baseAngle) - mPointOffetScreenY)
                  - mMarginBottom);

          child.layout(l, t, l + cWidth, t + cHeight);
          child.setVisibility(View.GONE);
      }
  }

  ```

这里的原理就是前面分析的，只是多了个marginBottom值，需要注意处理下。下面`onDraw`方法实现就非常简单了：

```java
@Override
   protected void onDraw(Canvas canvas) {
       super.onDraw(canvas);
       canvas.drawCircle(mDrawX, mDrawingY, mDrawingBackgroundRadius, mPaint);
   }
   ```
绘制的关键是动画控制以上这些变量去动态绘制。

下面是动画的添加：

```java
private void toggleMenuItemAnim(int pos, int duration) {

        final View childView = getChildAt(pos);

        TranslateAnimation tranlateAnim = null;
        AlphaAnimation alphaAnimation = null;
        // to open
        if (mStatus == Status.CLOSE) {
            tranlateAnim = new TranslateAnimation(0, 0, childView.getMeasuredHeight(), 0);
            tranlateAnim.setInterpolator(new DecelerateInterpolator());

            alphaAnimation = new AlphaAnimation(0.0f, 1.0f);
            alphaAnimation.setDuration(duration / 2);

            childView.setClickable(true);
            childView.setFocusable(true);
        } else {
            tranlateAnim = new TranslateAnimation(0, 0, 0, childView.getMeasuredHeight());
            tranlateAnim.setInterpolator(new DecelerateInterpolator());

            alphaAnimation = new AlphaAnimation(1.0f, 0.0f);
            alphaAnimation.setDuration(duration / 2);

        }
        tranlateAnim.setDuration(duration);

        AnimationSet animset = new AnimationSet(false);
        animset.addAnimation(tranlateAnim);
        animset.addAnimation(alphaAnimation);

        animset.setFillAfter(true);
        childView.startAnimation(animset);

    }

    /**
     * 添加menuItem的点击动画
     */
    private void menuItemClickAnim(int pos) {
        isAnimRunning = true;
        if (mOnAnimationRunningListener != null) {
            mOnAnimationRunningListener.onAnimationStart();
        }
        for (int i = 1; i < getChildCount(); i++) {

            View childView = getChildAt(i);
            if (i == pos) {
                childView.startAnimation(scaleBigAnim(300, childView, pos - 1));
            } else {
                childView.startAnimation(scaleSmallAnim(300, childView));
            }
        }
    }

    private Animation scaleSmallAnim(int duration, final View child) {

        AnimationSet animationSet = new AnimationSet(true);

        ScaleAnimation scaleAnim = new ScaleAnimation(1.0f, 0.0f, 1.0f, 0.0f,
                Animation.RELATIVE_TO_SELF, 0.5f, Animation.RELATIVE_TO_SELF,
                0.5f);
        AlphaAnimation alphaAnim = new AlphaAnimation(1f, 0.0f);
        alphaAnim.setFillAfter(true);
        animationSet.addAnimation(scaleAnim);
        animationSet.addAnimation(alphaAnim);
        animationSet.setDuration(duration);

        return animationSet;

    }

    private Animation scaleBigAnim(int duration, final View child, final int position) {
        AnimationSet animationSet = new AnimationSet(true);

        ScaleAnimation scaleAnim = new ScaleAnimation(1.0f, 4.0f, 1.0f, 4.0f,
                Animation.RELATIVE_TO_SELF, 0.5f, Animation.RELATIVE_TO_SELF,
                0.5f);
        AlphaAnimation alphaAnim = new AlphaAnimation(1.0f, 0.0f);

        animationSet.addAnimation(scaleAnim);
        animationSet.addAnimation(alphaAnim);

        animationSet.setDuration(duration);
        animationSet.setFillAfter(true);
        animationSet.setAnimationListener(new Animation.AnimationListener() {
            @Override
            public void onAnimationStart(Animation animation) {

            }

            @Override
            public void onAnimationEnd(Animation animation) {
                if (mOnMenuItemClickListener != null) {
                    mOnMenuItemClickListener.onClick(child, position);
                }
            }

            @Override
            public void onAnimationRepeat(Animation animation) {

            }
        });
        return animationSet;

    }


    private void toggleBackgroundAnim(int duration) {
        final ValueAnimator animator;
        if (mStatus == Status.OPEN) {
            animator = ValueAnimator.ofFloat(1, 0);
        } else {
            animator = ValueAnimator.ofFloat(0, 1);
        }
        animator.setDuration(duration);
        animator.setInterpolator(new DecelerateInterpolator());
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                // 变化区间设在0~1  方便映射到其他区间
                float cVal = (float) animation.getAnimatedValue();
                setBackgroundColor(Color.argb((int) (cVal * 180), 255, 255, 255));
                mDrawingBackgroundRadius = (int) (mBackgroundRadius * cVal);
                mDrawingY = (int) (mStartDrawingY + mDrawYOffset * cVal);
                invalidate();
                if (mOnAnimationRunningListener != null) {
                    mOnAnimationRunningListener.onAnimationRunning(cVal);
                }
            }
        });
        animator.addListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {
                if (mStatus == Status.CLOSE) {
                    for (int i = 1; i < getChildCount(); i++) {
                        getChildAt(i).clearAnimation();
                        getChildAt(i).setFocusable(false);
                        getChildAt(i).setClickable(false);
                        getChildAt(i).setVisibility(GONE);
                    }
                }
                isAnimRunning = false;
                if (mOnAnimationRunningListener != null) {
                    mOnAnimationRunningListener.onAnimationEnd();
                }
            }

            @Override
            public void onAnimationCancel(Animator animation) {

            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }
        });
        animator.start();
        changeStatus();
    }
```
从代码中可以看出，除了我之前说的menuitem动画和background动画外，我还写了菜单选中的动画效果，为了让菜单点击时体验更好，也是属性动画。

* *注意，在动画完成之后要记得调用每个item的`clearAnimation`方法*

事件的处理也比较简单，只需要处理DOWN事件，在菜单打开状态时，触摸任意地方致菜单关闭即可，当菜单关闭状态，除了按钮其他不响应事件也不拦截事件，这样一来，就可以将菜单铺满在任意界面，在使用菜单的同时不会影响其他交互，但一搬放在首页的tabBar上吧。只需处理一个`onTouch`方法即可：
```java
@Override
  public boolean onTouchEvent(MotionEvent event) {
      if (mStatus == Status.OPEN) {
          if (!isAnimRunning) {
              toggleMenu(mAnimDuration);
          }
          return true;
      }
      return false;
  }
```

对外提供的能力主要是，菜单选中监听，动画过程监听，状态判断等
```java
    public View getToggleView() {
        return mButton;
    }

    public void setOnMenuItemClickListener(OnMenuItemClickListener listener) {
        this.mOnMenuItemClickListener = listener;
    }

    public void setmOnAnimationRunningListener(OnAnimationRunningListener listener) {
        mOnAnimationRunningListener = listener;
    }

    public boolean isOpen() {
        return mStatus == Status.OPEN;
    }
```

到此为止，整个控件的实现基本完成，很简单不过500多行代码，但是涉及的东西还是比较全面的，也是让我有所收获，所以分享出来。看到这可能有人不禁吐槽，这么个东西写这么长的文，真是……哈哈哈，这是态度问题（认真脸）。还有，这只是我的个人见解，如有人有更简单更完美的思路，欢迎交流。

完整代码[在这](https://github.com/Ethan-9606/BottomMenu){:target="_blank"}，欢迎 **star**   **issue**
