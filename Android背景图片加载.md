对大多数Android的开发者来说，最经常的操作莫过于对界面进行布局，View中背景图片的加载是最经常做的。但是我们很少关注这个过程，这篇文章主要解析view中背景图片加载的流程。了解view中背景图片的加载（资源的加载）可以让我们对资源加载的过程进行一些优化，另外当需要进行整个应用的换肤时，也可以更得心应手。

View图片的加载，我们最常见的就是通过在XML文件当中进行drawable的设置，然后让Android系统帮我们完成，或者手动写代码加载成Bitmap，然后加载到View上。这篇文章主要分析Android在什么时候以及怎么帮我们完成背景图片的加载的，那么我们就从Activity.setContentView还是LayoutInflater.inflate(...)方法开始分析。

不管是从[Activity.setContentView(...)](http://developer.android.com/reference/android/app/Activity.html#setContentView(android.view.View))还是[LayoutInflater.inflate(...)](http://developer.android.com/reference/android/view/LayoutInflater.html)方法进行View的初始化，最终都会到达LayoutInflater.inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)这个方法中。在这里我们主要关注View的背景图片加载，对于XML如何解析和加载就放过了。

``` java
    public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context)mConstructorArgs[0];
            mConstructorArgs[0] = mContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    View temp;
                    if (TAG_1995.equals(name)) {
                        temp = new BlinkLayout(mContext, attrs);
                    } else {
                        temp = createViewFromTag(root, name, attrs);
                    }

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }
                     // Inflate all children under temp
                    rInflate(parser, temp, attrs, true);
                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (IOException e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                        + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }

            return result;
        }
    }
```

上面这么长一串代码，其实思路很清晰，就是针对XML文件进行解析，然后根据XML解析出的每一个节点进行View的初始化，紧接着将View的Layout参数设置到View上，然后将View添加到它的父控件上。
为了了解View是怎么被加载出来的，我们只需要了解
           
``` java         
	temp = createViewFromTag(root, name, attrs);
```
跟进去看看。

``` java
    /*
     * default visibility so the BridgeInflater can override it.
     */
    View createViewFromTag(View parent, String name, AttributeSet attrs) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        if (DEBUG) System.out.println("******** Creating view: " + name);

        try {
            View view;
            if (mFactory2 != null) view = mFactory2.onCreateView(parent, name, mContext, attrs);
            else if (mFactory != null) view = mFactory.onCreateView(name, mContext, attrs);
            else view = null;

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, mContext, attrs);
            }
            
            if (view == null) {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    view = createView(name, null, attrs);
                }
            }

            if (DEBUG) System.out.println("Created view is: " + view);
            return view;

        } catch (InflateException e) {
            throw e;

        } catch (ClassNotFoundException e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;

        } catch (Exception e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;
        }
    }
```

上面代码的重点在于try...Catch里的内容。try包起来的东西就是对View进行初始化，注意到上面代码中有几个Factory，这些Factory可以在View进行初始化，也就是说其实我们可以在这里干预View的初始化。从上面代码我们可以知道，如果我们自定义了一个Factory，那么当前要初始化的View会优先被我们自定义的Factory初始化，而不通过系统默认的Factory初始化。那么如果我们要自定义Factory，应该在哪里定义呢？容易想到，Factory必须要赶在资源加载前自定义完成，所以我们应该在onCreate(...)的this.setContentView(...)之前设置LayoutInflater.Factory。
``` java
		getLayoutInflater().setFactory(factory);
```
接下来我们看到上面函数里面的

``` java
 	if (-1 == name.indexOf('.')) {
        view = onCreateView(parent, name, attrs);
    } else {
        view = createView(name, null, attrs);
    }
```

这段函数就是对View进行初始化，有两种情况，一种是系统自带的View，它在

``` java
	if (-1 == name.indexOf('.'))
```

这里面进行初始化，因为如果是系统自带的View，传入的那么一般不带系统的前缀"android.view."。另一个分支初始化的是我们自定义的View。我们跟进onCreateView看看。

``` java
 	protected View onCreateView(String name, AttributeSet attrs)
            throws ClassNotFoundException {
        return createView(name, "android.view.", attrs);
    }

    public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;

        try {
            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                
                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                constructor = clazz.getConstructor(mConstructorSignature);
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);
                        
                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
            }

            Object[] args = mConstructorArgs;
            args[1] = attrs;

            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // always use ourselves when inflating ViewStub later
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(this);
            }
            return view;

        } catch (NoSuchMethodException e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class "
                    + (prefix != null ? (prefix + name) : name));
            ie.initCause(e);
            throw ie;

        } catch (ClassCastException e) {
            // If loaded class is not a View subclass
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Class is not a View "
                    + (prefix != null ? (prefix + name) : name));
            ie.initCause(e);
            throw ie;
        } catch (ClassNotFoundException e) {
            // If loadClass fails, we should propagate the exception.
            throw e;
        } catch (Exception e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class "
                    + (clazz == null ? "<unknown>" : clazz.getName()));
            ie.initCause(e);
            throw ie;
        }
    }
```
从onCreateView(...)中我们知道，其实createViewFromTag(...)中对View的初始化最终都是通过createView(...)这个函数进行初始化的，不同只在于系统控件需要通过onCreateView(...)加上前缀，以便类加载器(ClassLoader)正确地通过类所在的包初始化这个类。createView(...)这个函数的思路很清晰，不看catch里面的内容，try里面开头的两个分支就是用来将所要用的类构造函数提取出来，Android系统会对使用过的类构造函数进行缓存，因为像TextView这些常用的控件可能会被使用很多次。接下来，就是通过类构造函数对View进行初始化了。我们注意到传入构造函数的mConstructorArgs是一个包含两个元素的数组。

``` java
	final Object[] mConstructorArgs = new Object[2];
```

那么我们就很清楚了，它就是调用系统控件中对应两个参数的构造函数。为了方便，我们就从最基础的View进行分析。
``` java
 	public View(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public View(Context context, AttributeSet attrs, int defStyle) {
	    this(context);
	
	    TypedArray a = context.obtainStyledAttributes(attrs, com.android.internal.R.styleable.View,
	            defStyle, 0);
	
	    Drawable background = null;
	
	    int leftPadding = -1;
	    int topPadding = -1;
	    int rightPadding = -1;
	    int bottomPadding = -1;
	    int startPadding = UNDEFINED_PADDING;
	    int endPadding = UNDEFINED_PADDING;
	
	    int padding = -1;
	
	    int viewFlagValues = 0;
	    int viewFlagMasks = 0;
	
	    boolean setScrollContainer = false;
	
	    int x = 0;
	    int y = 0;
	
	    float tx = 0;
	    float ty = 0;
	    float rotation = 0;
	    float rotationX = 0;
	    float rotationY = 0;
	    float sx = 1f;
	    float sy = 1f;
	    boolean transformSet = false;
	
	    int scrollbarStyle = SCROLLBARS_INSIDE_OVERLAY;
	    int overScrollMode = mOverScrollMode;
	    boolean initializeScrollbars = false;
	
	    boolean leftPaddingDefined = false;
	    boolean rightPaddingDefined = false;
	    boolean startPaddingDefined = false;
	    boolean endPaddingDefined = false;
	
	    final int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;
	
	    final int N = a.getIndexCount();
		    for (int i = 0; i < N; i++) {
		        int attr = a.getIndex(i);
		        switch (attr) {
		            case com.android.internal.R.styleable.View_background:
		                background = a.getDrawable(attr);
		                break;
		            case com.android.internal.R.styleable.View_padding:
		                padding = a.getDimensionPixelSize(attr, -1);
		                mUserPaddingLeftInitial = padding;
		                mUserPaddingRightInitial = padding;
		                leftPaddingDefined = true;
		                rightPaddingDefined = true;
		                break;
			
			//省略一大串无关的函数
	}
```
由于我们只关注View中的背景图是怎么加载的，注意这个函数其实就是遍历AttributeSet attrs这个东西，然后对View的各个属性进行初始化。我们直接进入
``` java
	background = a.getDrawable(attr);
```
这里看看(TypedArray.getDrawable)。
``` java
    public Drawable getDrawable(int index) {
        final TypedValue value = mValue;
        if (getValueAt(index*AssetManager.STYLE_NUM_ENTRIES, value)) {
            if (false) {
                System.out.println("******************************************************************");
                System.out.println("Got drawable resource: type="
                                   + value.type
                                   + " str=" + value.string
                                   + " int=0x" + Integer.toHexString(value.data)
                                   + " cookie=" + value.assetCookie);
                System.out.println("******************************************************************");
            }
            return mResources.loadDrawable(value, value.resourceId);
        }
        return null;
    }
```
我们发现它调用mResources.loadDrawable(...)，进去看看。
``` java
    /*package*/ Drawable loadDrawable(TypedValue value, int id)
            throws NotFoundException {

        if (TRACE_FOR_PRELOAD) {
            // Log only framework resources
            if ((id >>> 24) == 0x1) {
                final String name = getResourceName(id);
                if (name != null) android.util.Log.d("PreloadDrawable", name);
            }
        }

        boolean isColorDrawable = false;
        if (value.type >= TypedValue.TYPE_FIRST_COLOR_INT &&
                value.type <= TypedValue.TYPE_LAST_COLOR_INT) {
            isColorDrawable = true;
        }
        final long key = isColorDrawable ? value.data :
                (((long) value.assetCookie) << 32) | value.data;

        Drawable dr = getCachedDrawable(isColorDrawable ? mColorDrawableCache : mDrawableCache, key);

        if (dr != null) {
            return dr;
        }

        Drawable.ConstantState cs = isColorDrawable
                ? sPreloadedColorDrawables.get(key)
                : (sPreloadedDensity == mConfiguration.densityDpi
                        ? sPreloadedDrawables.get(key) : null);
        if (cs != null) {
            dr = cs.newDrawable(this);
        } else {
            if (isColorDrawable) {
                dr = new ColorDrawable(value.data);
            }

            if (dr == null) {
                if (value.string == null) {
                    throw new NotFoundException(
                            "Resource is not a Drawable (color or path): " + value);
                }

                String file = value.string.toString();

                if (TRACE_FOR_MISS_PRELOAD) {
                    // Log only framework resources
                    if ((id >>> 24) == 0x1) {
                        final String name = getResourceName(id);
                        if (name != null) android.util.Log.d(TAG, "Loading framework drawable #"
                                + Integer.toHexString(id) + ": " + name
                                + " at " + file);
                    }
                }

                if (DEBUG_LOAD) Log.v(TAG, "Loading drawable for cookie "
                        + value.assetCookie + ": " + file);

                if (file.endsWith(".xml")) {
                    try {
                        XmlResourceParser rp = loadXmlResourceParser(
                                file, id, value.assetCookie, "drawable");
                        dr = Drawable.createFromXml(this, rp);
                        rp.close();
                    } catch (Exception e) {
                        NotFoundException rnf = new NotFoundException(
                            "File " + file + " from drawable resource ID #0x"
                            + Integer.toHexString(id));
                        rnf.initCause(e);
                        throw rnf;
                    }

                } else {
                    try {
                        InputStream is = mAssets.openNonAsset(
                                value.assetCookie, file, AssetManager.ACCESS_STREAMING);
        //                System.out.println("Opened file " + file + ": " + is);
                        dr = Drawable.createFromResourceStream(this, value, is,
                                file, null);
                        is.close();
        //                System.out.println("Created stream: " + dr);
                    } catch (Exception e) {
                        NotFoundException rnf = new NotFoundException(
                            "File " + file + " from drawable resource ID #0x"
                            + Integer.toHexString(id));
                        rnf.initCause(e);
                        throw rnf;
                    }
                }
            }
        }

        if (dr != null) {
            dr.setChangingConfigurations(value.changingConfigurations);
            cs = dr.getConstantState();
            if (cs != null) {
                if (mPreloading) {
                    if (verifyPreloadConfig(value, "drawable")) {
                        if (isColorDrawable) {
                            sPreloadedColorDrawables.put(key, cs);
                        } else {
                            sPreloadedDrawables.put(key, cs);
                        }
                    }
                } else {
                    synchronized (mTmpValue) {
                        //Log.i(TAG, "Saving cached drawable @ #" +
                        //        Integer.toHexString(key.intValue())
                        //        + " in " + this + ": " + cs);
                        if (isColorDrawable) {
                            mColorDrawableCache.put(key, new WeakReference<Drawable.ConstantState>(cs));
                        } else {
                            mDrawableCache.put(key, new WeakReference<Drawable.ConstantState>(cs));
                        }
                    }
                }
            }
        }

        return dr;
    }
```
就是这个函数了，所有View的背景的加载都在这里了。这个函数的逻辑就比较复杂了，大体说来就是根据背景的类型（纯颜色、定义在XML文件中的，或者是一张静态的背景），如果缓存里面有，就直接用缓存里的。

总结一下，经过上面的分析，我们知道了，Android就是在Activity.setContentView(...)中为我们进行资源文件的加载，精确到具体的函数的话，资源文件的加载就是在每一个被初始化的View的构造函数中进行加载的。