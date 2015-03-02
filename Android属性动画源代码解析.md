本文假定你已经对[属性动画](http://developer.android.com/reference/android/animation/ObjectAnimator.html)有了一定的了解，至少使用过属性动画。下面我们就从属性动画最简单的使用开始。

``` java
    ObjectAnimator
      .ofInt(target,propName,values[])
      .setInterpolator(LinearInterpolator)
      .setEvaluator(IntEvaluator)
      .setDuration(500)
      .start();
```
相信这段代码对你一定不陌生，代码中有几个地方是本文中将要重点关注的，[setInterpolator(...)](http://developer.android.com/reference/android/animation/ValueAnimator.html#setInterpolator(android.animation.TimeInterpolator))、[setEvaluator(...) ](http://developer.android.com/reference/android/animation/ValueAnimator.html#setEvaluator(android.animation.TypeEvaluator))、[setDuration(...)](http://developer.android.com/reference/android/animation/ValueAnimator.html#setDuration(long))在源代码中是如何被使用的。另外，我们也将重点关注Android中属性动画是如何一步步地实现动画效果的（精确到每一帧(frame)）。最后啰嗦几句，本文中使用的代码是Android 4.2.2。

上面代码的作用就是生成一个属性动画，根据ofInt()我们知道只是一个属性值类型为Int的View的动画。先放过其他的函数，从ObjectAnimator的[start（）](http://developer.android.com/reference/android/animation/ObjectAnimator.html#start())函数开始。
    
``` java
    public void  start() {
    	//...省略不必要代码 
    	super.start();
    }
``` 
从代码中我们知道，它调用了父类的[start()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.2_r1/android/animation/ValueAnimator.java#ValueAnimator.start%28%29)方法，也就是[ValueAnimator.start()](http://developer.android.com/reference/android/animation/ValueAnimator.html#start())。这个方法内部又调用了自身类内部的start(boolean playBackwards)方法。

``` java
    /**
	 * 方法简要介绍：
     * 这个方法开始播放动画。这个start()方法使用一个boolean值playBackwards来判断是否需
     * 要回放动画。这个值通常为false，但当它从reverse()方法被调用的时候也
	 * 可能为true。有一点需要注意的是，这个方法必须从UI主线程调用。
     */
    private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        mPlayingBackwards = playBackwards;
        mCurrentIteration = 0;
        mPlayingState = STOPPED;
        mStarted = true;
        mStartedDelay = false;
        AnimationHandler animationHandler = getOrCreateAnimationHandler();
        animationHandler.mPendingAnimations.add(this);
        if (mStartDelay == 0) {
            // 在动画实际运行前，设置动画的初始值
            setCurrentPlayTime(0);
            mPlayingState = STOPPED;
            mRunning = true;
            notifyStartListeners();
        }
        animationHandler.start();
    }
```

**对代码中几个值的解释**：

- mPlayingStated代表当前动画的状态。用于找出什么时候开始动画(if state == STOPPED)。当然也用于在animator被调用了[cancel()](http://developer.android.com/reference/android/animation/ValueAnimator.html#cancel())或 [end()](http://developer.android.com/reference/android/animation/ValueAnimator.html#end())在动画的最后一帧停止它。可能的值为STOPPED, RUNNING, SEEKED.
- mStarted是Animator中一个额外用于标识播放状态的值，用来指示这个动画是否需要延时执行。
- mStartedDelay指示这个动画是否已经从startDelay中开始执行。
- [AnimationHandler](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.2_r1/android/animation/ValueAnimator.java#ValueAnimator.AnimationHandler) animationHandler 是一个实现了Runnable接口的ValueAnimator内部类，暂时先放过，后面我们会具体谈到。

从上面这段代码中，我们了解到一个ValueAnimator有它自己的状态(STOPPED, RUNNING, SEEKED)，另外是否延时也影响ValueAnimator的执行。代码的最后调用了animationHandler.start()，看来动画就是从这里启动的。别急，我们还没初始化ValueAnimator呢，跟进setCurrentPlayTime(0)看看。

``` java
    public void setCurrentPlayTime(long playTime) {
        initAnimation();
        long currentTime = AnimationUtils.currentAnimationTimeMillis();
        if (mPlayingState != RUNNING) {
            mSeekTime = playTime;
            mPlayingState = SEEKED;
        }
        mStartTime = currentTime - playTime;
        doAnimationFrame(currentTime);
    }
```

这个函数在animation开始前，设置它的初始值。这个函数用于设置animation进度为指定时间点。playTime应该介于0到animation的总时间之间，包括animation重复执行的时候。如果animation还没有开始，那么它会等到被设置这个时间后才开始。如果animation已经运行，那么setCurrentTime()会将当前的进度设置为这个值，然后从这个点继续播放。

接下来让我们看看initAnimation()

``` java
    void initAnimation() {
        if (!mInitialized) {
            int numValues = mValues.length;
            for (int i = 0; i < numValues; ++i) {
                mValues[i].init();
            }
            mInitialized = true;
        }
    }
```

这个函数一看就觉得跟初始化动画有关。这个函数在处理动画的第一帧前就会被调用。如果startDelay不为0，这个函数就会在就会在延时结束后调用。它完成animation最终的初始化。


那么mValues是什么呢？还记得我们在文章的开头介绍ObjectAnimator的使用吧？还有一个ofInt(T target, Property<T, Integer> property, int... values)方法没有介绍。官方文档中对这个方法的解释是：构造并返回一个在int类型的values数值之间ObjectAnimator对象。当values只有一个值的时候，这个值就作为animator的终点值。如果有两个值的话，那么这两个值就作为开始值和结束值。如果有超过两个以上的值的话，那么这些值就作为开始值，作为animator运行的中间值，以及结束值。这些值将均匀地分配到animator的持续时间。

## 先中断ObjectAnimator.start()流程的分析，回到开头ObjectAnimator.ofInt(...)  ##

接下来让我们深入ofInt(...)的内部看看。

``` java
    public static ObjectAnimator ofInt(Object target, String propertyName, int... values) {
        ObjectAnimator anim = new ObjectAnimator(target, propertyName);
        anim.setIntValues(values);
        return anim;
    }
```

对这个函数的解释:

- target 就是将要进行动画的对象
- propertyName 就是这个对象属性将要进行动画的属性名
- values 一组值。随着时间的推移，动画将根据这组值进行变化。

再看看anim.setIntValues这个函数

``` java
    public void setIntValues(int... values) {
        if (mValues == null || mValues.length == 0) {
            // No values yet - this animator is being constructed piecemeal. Init the values with
            // whatever the current propertyName is
            if (mProperty != null) {
                setValues(PropertyValuesHolder.ofInt(mProperty, values));
            } else {
                setValues(PropertyValuesHolder.ofInt(mPropertyName, values));
            }
        } else {
            super.setIntValues(values);
        }
    }
```

一开始的时候，mProperty肯定还没有初始化，我们进去setValues(PropertyValuesHolder.ofInt(mPropertyName, values))看看。这里涉及到[PropertyValuesHolder](http://developer.android.com/reference/android/animation/PropertyValuesHolder.html)这个类。PropertyValuesHolder这个类拥有关于属性的信息和动画期间需要使用的值。PropertyValuesHolder对象可以用来和ObjectAnimator或ValueAnimator一起创建可以并行操作不PropertyValuesHolder同属性的animator。

那么PropertyValuesHolder.ofInt()是干嘛用的呢？它通过传入的属性和values来构造并返回一个特定类型的PropertyValuesHolder对象（在这里是[IntPropertyValuesHolder](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.2_r1/android/animation/PropertyValuesHolder.java#PropertyValuesHolder.IntPropertyValuesHolder)类型）。

``` java
    public static PropertyValuesHolder ofInt(String propertyName, int... values) {
        return new IntPropertyValuesHolder(propertyName, values);
    }
```

在IntPropertyValuesHolder内部

``` java
    public IntPropertyValuesHolder(String propertyName, int... values) {
        super(propertyName);
        setIntValues(values);
    }

    @Override
    public void setIntValues(int... values) {
        super.setIntValues(values);
        mIntKeyframeSet = (IntKeyframeSet) mKeyframeSet;
    }
```

跳转到父类（PropertyValuesHolder）的setIntValues
``` java
    public void setIntValues(int... values) {
        mValueType = int.class;
        mKeyframeSet = KeyframeSet.ofInt(values);
    }
```

这个函数其实跟我们前面介绍到的 PropertyValueHolder的构造函数相似，它就是设置动画过程中需要的值。如果只有一个值，那么这个值就假定为animator的终点值，动画的初始值会自动被推断出来,通过对象的getter方法得到。当然，如果所有值都为空，那么同样的这些值也会在动画开始的时候也会自动被填上。这套自动推断填值的机制只在PropertyValuesHolder对象跟ObjectAnimator一起使用的时候才有效，并且有一个能从propertyName自动推断出的getter方法这些条件都成立的时候才能用，不然PropertyValuesHolder没有办法决定这些值是什么。
接下来我们看到KeyframeSet.ofInt(values)方法。[KeyframeSet](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.2_r1/android/animation/KeyframeSet.java#KeyframeSet)这个类持有Keyframe的集合，在一组给定的animator的关键帧(keyframe)中会被ValueAnimator用来计算值。这个类的访问权限为包可见，因为这个类实现Keyframe怎么被存储和使用的具体细节，外部不需要知道。

接下来我们看看[KeyframeSet.ofInt(values)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.2_r1/android/animation/KeyframeSet.java#KeyframeSet.ofInt%28int%5B%5D%29)方法。

``` java
    public static KeyframeSet ofInt(int... values) {
        int numKeyframes = values.length;
        IntKeyframe keyframes[] = new IntKeyframe[Math.max(numKeyframes,2)];
        if (numKeyframes == 1) {
            keyframes[0] = (IntKeyframe) Keyframe.ofInt(0f);
            keyframes[1] = (IntKeyframe) Keyframe.ofInt(1f, values[0]);
        } else {
            keyframes[0] = (IntKeyframe) Keyframe.ofInt(0f, values[0]);
            for (int i = 1; i < numKeyframes; ++i) {
                keyframes[i] = (IntKeyframe) Keyframe.ofInt((float) i / (numKeyframes - 1), values[i]);
            }
        }
        return new IntKeyframeSet(keyframes);
    }
```

在这个方法里面，我们看见从最开始的ObjectAnimator.ofInt(target,propName,values[])，也就是我们在文章的开头使用系统提供的动画初始化函数中传入的int数组，在这里得到了具体的使用。先不关心具体的使用Keyframe.ofInt(...)。从这里我们就可以知道原来Android SDK通过传入的int[]的长度来决定animator中每个帧(frame)的值。具体的对传入的int[]的使用可以参考文章里面对ObjectAnimator.ofInt(...)的介绍。在KeyframeSet.ofInt这个函数的最后一句话使用了IntKeyframeSet的构造函数来初始化这些Keyframe。

``` java
    public  IntKeyframeSet(IntKeyframe... keyframes) {
         super(keyframes);
    }
```
在IntKeyframeSet的构造函数中又调用父类KeyframeSet的构造函数来实现。

``` java
    public KeyframeSet(Keyframe... keyframes) {
        mNumKeyframes = keyframes.length;
        mKeyframes = new ArrayList<Keyframe>();
        mKeyframes.addAll(Arrays.asList(keyframes));
        mFirstKeyframe = mKeyframes.get(0);
        mLastKeyframe = mKeyframes.get(mNumKeyframes - 1);
        mInterpolator = mLastKeyframe.getInterpolator();
    }
```

从这个构造函数中我们又可以了解到刚刚初始化后的Keyframe数组的第一项和最后一项（也就是第一帧和最后一帧）得到了优先的待遇，作为在KeyframeSet中的字段，估计是为了后面计算动画开始和结束的时候方便。

**小结ObjectValue、PropertyValueHolder、KeyframeSet的关系**

我们绕了很久，不知道是否把你弄晕了，这里稍稍总结一下。我们就不从调用的顺序一步步分析下来了，太长了。我直接说说ObjectValue、PropertyValueHolder、KeyframeSet之间的关系。这三个类比较有特点的地方，ObjectAnimator无疑是对的API接口，ObjectAnimator持有PropertyValuesHolder作为存储关于将要进行动画的具体对象(通常是View类型的控件)的属性和动画期间需要的值。而PropertyValueHolder又使用KeyframeSet来保存animator从开始到结束期间关键帧的值。这下子我们就了解animator在执行期间用来存储和使用的数据结构。废话一下，从PropertyValueHolder、KeyframeSet这个两个类的源代码来看，这三个类的API的设计挺有技巧的，他们都是通过将具有特定类型的实现作为一个大的概况性的类的内部实现，通过这个大的抽象类提供对外的API（例如，PropertyValuesHolder.ofInt(...)的实现）。

## 回到ObjectAnimator.start()流程的分析 ##

不知道是否把你上面你是否能清楚，反正不太影响下面对ObjectAnimator.start()流程的分析。从上面一段的分析我们了解到ValueAnimator.initAnimation()中的mValue是 PropertyValuesHolder类型的东西。在initAnimation()里mValues[i].init()初始化它们的估值器Evaluator

``` java
    void init() {
        if (mEvaluator == null) {
            // We already handle int and float automatically, but not their Object
            // equivalents
            mEvaluator = (mValueType == Integer.class) ? sIntEvaluator :
                    (mValueType == Float.class) ? sFloatEvaluator :
                    null;
        }
        if (mEvaluator != null) {
            // KeyframeSet knows how to evaluate the common types - only give it a custom
            // evaluator if one has been set on this class
            mKeyframeSet.setEvaluator(mEvaluator);
        }
    }
```

mEvaluator当然也可以使用ObjectAnimator.setEvaluator(...)传入；为空时，SDK根据mValueType为我们初始化特定类型的Evaluator。这样我们的初始化就完成了。接下来，跳出initAnimation()回到
setCurrentPlayTime(...)

``` java
    public void setCurrentPlayTime(long playTime) {
        initAnimation();
        long currentTime = AnimationUtils.currentAnimationTimeMillis();
        if (mPlayingState != RUNNING) {
            mSeekTime = playTime;
            mPlayingState = SEEKED;
        }
        mStartTime = currentTime - playTime;
        doAnimationFrame(currentTime);
    }
```

**对animator三种状态STOPPED、RUNNING、SEEKED的解释**

-  		static final int STOPPED    = 0; // 还没开始播放
-       static final int RUNNING    = 1; // 正常播放中
-       static final int SEEKED     = 2; // 定位到一些时间值(Seeked to some time value)

对mSeekedTime、mStartTime的解释
- mSeekedTime 当setCurrentPlayTime()被调用的时候设置。如果为负数，animator还没能定位到一个值。
- mStartTime 第一次在animation.animateFrame()方法调用时使用。这个时间在第二次调用animateFrame()时用来确定运行时间(以及运行的分数值)

setCurrentPlayTime(...)中doAnimationFrame(currentTime) 之前的代码其实都是对Animator的初始化。看来doAnimator(...)就是真正处理动画帧的函数了。这个函数主要主要用来处理animator中的一帧，并在有必要的时候调整animator的开始时间。

``` java
    final boolean doAnimationFrame(long frameTime) {
		//对animator的开始时间和状态进行调整
        if (mPlayingState == STOPPED) {
            mPlayingState = RUNNING;
            if (mSeekTime < 0) {
                mStartTime = frameTime;
            } else {
                mStartTime = frameTime - mSeekTime;
                // Now that we're playing, reset the seek time
                mSeekTime = -1;
            }
        }
        // The frame time might be before the start time during the first frame of
        // an animation.  The "current time" must always be on or after the start
        // time to avoid animating frames at negative time intervals.  In practice, this
        // is very rare and only happens when seeking backwards.
        final long currentTime = Math.max(frameTime, mStartTime);
        return animationFrame(currentTime);
    }
```

看来这个函数就是调整了一些参数，真正的处理函数还在animationFrame(...)中。我们跟进去看看。

``` java
    boolean animationFrame(long currentTime) {
        boolean done = false;
        switch (mPlayingState) {
        case RUNNING:
        case SEEKED:
            float fraction = mDuration > 0 ? (float)(currentTime - mStartTime) / mDuration : 1f;
            if (fraction >= 1f) {
                if (mCurrentIteration < mRepeatCount || mRepeatCount == INFINITE) {
                    // Time to repeat
                    if (mListeners != null) {
                        int numListeners = mListeners.size();
                        for (int i = 0; i < numListeners; ++i) {
                            mListeners.get(i).onAnimationRepeat(this);
                        }
                    }
                    if (mRepeatMode == REVERSE) {
                        mPlayingBackwards = mPlayingBackwards ? false : true;
                    }
                    mCurrentIteration += (int)fraction;
                    fraction = fraction % 1f;
                    mStartTime += mDuration;
                } else {
                    done = true;
                    fraction = Math.min(fraction, 1.0f);
                }
            }
            if (mPlayingBackwards) {
                fraction = 1f - fraction;
            }
            animateValue(fraction);
            break;
        }

        return done;
    }
```

这个内部函数对给定的animation的一个简单的动画帧进行处理。currentTime这个参数是由定时脉冲（先不要了解这个定时脉冲是什么，后面我们会涉及）通过handler发送过来的（当然也可能是初始化的时候，被程序调用的，就像我们的分析过程一样），它用于计算animation已运行的时间，以及已经运行分数值。这个函数的返回值标识这个animation是否应该停止（在运行时间超过animation应该运行的总时长的时候，包括重复次数超过的情况）。

我们可以把这个函数里面的fraction简单地理解成animator的进度条的当前的位置。*if (fraction >= 1f)* 注意到函数里面的这句话，当animator开始需要重复执行的时候，那么就需要执行这个if判断里面的东西，这里面主要就是记录和改变重复执行animator的一些状态和变量。为了不让这篇文章太复杂，我们这里就不进行分析了。通过最简单的animator只执行一次的情况来分析。那么接下来就应该执行animateValue(fraction)了。

``` java
    void animateValue(float fraction) {
        fraction = mInterpolator.getInterpolation(fraction);
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].calculateValue(fraction);
        }
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }
```

在每一个animator的帧，这个函数都会被调用，结合传入的参数：已运行时间分数(fraction)。这个函数将已经运行的分数转为interpolaterd分数，然后转化成一个可用于动画的值（通过evaluator、这个函数通常在animation update的时候调用，但是它也可能在end()函数调用的时候被调用，用来设置property的最终值）。

在这里我们需要理清一下Interpolaterd和evaluator之间的关系。

- Interpolator:用来定义animator变化的速率。它让基础的动画效果(渐变、拉伸、平移、旋转)有加速、减速、重复等效果。在源代码中，其实interpolation的作用就是根据某一个时间点来计算它的播放时间分数,具体见[官方文档](http://developer.android.com/guide/topics/graphics/prop-animation.html)。
- evaluator： evaluator全部继承至[TypeEvaluator接口](http://developer.android.com/reference/android/animation/TypeEvaluator.html)，它只有一个[evaluate()方法](http://developer.android.com/reference/android/animation/TypeEvaluator.html#evaluate(float, T, T))。它用来返回你要进行动画的那个属性在当前时间点所需要的属性值。

我们可以把动画的过程想象成是一部电影的播放，电影的播放中有进度条，Interpolator就是用来控制电影播放频率，也就是快进快退要多少倍速。然后Evaluator根据Interpolator提供的值计算当前播放电影中的哪一个画面，也就是进度条要处于什么位置。

**这个函数分三步**:

1. 通过Interpolator计算出动画运行时间的分数
1. 变量ValueAnimator中的mValues[i].calculateValue(fraction)（也就是 PropertyValuesHolder对象数组）计算当前动画的值
1. 调用animation的[onAnimationUpdate(...)](http://developer.android.com/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html)通知animation更新的消息 

``` java
		//PropertyValuesHolder.calculateValue(...)
	    void calculateValue(float fraction) {
	        mAnimatedValue = mKeyframeSet.getValue(fraction);
	    }

		//mKeyframeSet.getValue
	    public Object getValue(float fraction) {

        // Special-case optimization for the common case of only two keyframes
        if (mNumKeyframes == 2) {
            if (mInterpolator != null) {
                fraction = mInterpolator.getInterpolation(fraction);
            }
            return mEvaluator.evaluate(fraction, mFirstKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        if (fraction <= 0f) {
            final Keyframe nextKeyframe = mKeyframes.get(1);
            final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = mFirstKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /
                (nextKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, mFirstKeyframe.getValue(),
                    nextKeyframe.getValue());
        } else if (fraction >= 1f) {
            final Keyframe prevKeyframe = mKeyframes.get(mNumKeyframes - 2);
            final TimeInterpolator interpolator = mLastKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = prevKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /
                (mLastKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        Keyframe prevKeyframe = mFirstKeyframe;
        for (int i = 1; i < mNumKeyframes; ++i) {
            Keyframe nextKeyframe = mKeyframes.get(i);
            if (fraction < nextKeyframe.getFraction()) {
                final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
                if (interpolator != null) {
                    fraction = interpolator.getInterpolation(fraction);
                }
                final float prevFraction = prevKeyframe.getFraction();
                float intervalFraction = (fraction - prevFraction) /
                    (nextKeyframe.getFraction() - prevFraction);
                return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                        nextKeyframe.getValue());
            }
            prevKeyframe = nextKeyframe;
        }
        // shouldn't reach here
        return mLastKeyframe.getValue();
    }
```

我们先只关注if (mNumKeyframes == 2)这种情况，因为这种情况最常见。还记的我们使用ObjectAnimator.ofInt(...)传入的int[]数组吗？我们一般就传入动画开始值和结束值，也就是这里的mNumKeyframes ==2 的情况。这里通过mEvaluator来计算，我们看看代码（[IntEvaluator.evaluate(...)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.2_r1/android/animation/IntEvaluator.java#IntEvaluator)的代码）。

``` java
    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }
```

mEvaluator.evaluate(...)计算后，我们就返回到ValueAnimator.animateValue(...)中，再回退到ValueAnimator.setCurrentPlayTime(...)。最后回到ValueAnimator.start(boolean playBackwards)。终于解析完了setCurrentPlayTime(...)这个函数，总结一下：这个函数主要用来初始化动画的值，当然这个初始化比我们想象中的复杂多了，它主要通过PropertyValuesHolder、Evaluator、Interpolator来进行值的初始化。PropertyValueHolder又通过KeyframeSet来存储需要的值。

## 我们回到文章开头介绍的ValueAnimator.start(boolean playBackwards) ##

``` java
    private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        mPlayingBackwards = playBackwards;
        mCurrentIteration = 0;
        mPlayingState = STOPPED;
        mStarted = true;
        mStartedDelay = false;
        AnimationHandler animationHandler = getOrCreateAnimationHandler();
        animationHandler.mPendingAnimations.add(this);
        if (mStartDelay == 0) {
            // This sets the initial value of the animation, prior to actually starting it running
            setCurrentPlayTime(0);
            mPlayingState = STOPPED;
            mRunning = true;
            notifyStartListeners();
        }
        animationHandler.start();
    }
```

在setCurrentPlayTime(0)后，紧接着就通过notifyStartListeners()通知animation启动的消息。最后通过animationHandler.start()去执行。animationHandler是一个AnimationHandler类型的对象，它实现了runable接口。

``` java
		//AnimationHandler.start()
        public void start() {
            scheduleAnimation();
        }
		//AnimationHandler.scheduleAnimation()
        private void scheduleAnimation() {
            if (!mAnimationScheduled) {
                mChoreographer.postCallback(Choreographer.CALLBACK_ANIMATION, this, null);
                mAnimationScheduled = true;
            }
        }

        // Called by the Choreographer.
        @Override
        public void run() {
            mAnimationScheduled = false;
            doAnimationFrame(mChoreographer.getFrameTime());
        }
```

mHandler.start()最终就是通过mChoreographer.发送给UI系统。这个过程比较复杂，这里不介绍。我们仅仅需要知道，动画中的一帧通过这种方式发送给UI系统后，在UI系统执行完一帧后，又会回调AnimationHandler.run()。那么其实这个过程就相当于，AnimationHandler.start（）开始第一次动画的执行→UI系统执行AnimationHandler.run()→UI系统执行完后，回调相关函数→再执行AnimationHandler.run().可以理解为AnimationHandler.run()会一直调用自身多次(当然这是由UI系统驱动的),直至动画结束。


这个过程比较复杂，如果你感兴趣，可以关注我的下一篇博客。