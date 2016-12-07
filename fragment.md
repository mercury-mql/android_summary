#Fragment

***

## 生命周期

Fragment必须内嵌在Activity中，并且Fragment的生命周期受到它所在的Activity的影响。

Fragment可以作为Activity UI的一部分，也可以设置为无UI的Fragment，从而作为Activity的不可见工作者（invisible worker）。

![](images\fragment_lifecycle.png) 

![](images/activity_fragment_lifecycle.png)

**注意： 当Fragment处于活动（active）状态时：**

- 如果用户回退、replace、remove此Fragment，会依次调用onPause-onStop-onDestroyView-onDestroy-onDetach
- 如果用户先将Fragment放入回退栈，然后又remove、replace该Fragment时，会依次调用onPause-onStop-onDestroyView，当该Fragment从回退栈返回时，会调用onCreateView


### 1. onInflate

<pre><code>
public void onInflate (Context context, AttributeSet attrs, Bundle savedInstanceState)
</code></pre>

当Fragment用作Activity UI的一部分，在布局填充时被调用。比较典型的：当在Activity中调用setContentView时，在通过layout中的<fragment\>标签创建Fragment后，此方法被立即调用。

**注意：此方法在onAttach之前被调用。**

此方法在fragment每次被填充时都会被调用，包括使用已保存的状态来创建新实例时。要注意在每次被调用时都要解析参数，从而能够应对不同的configuration。


<pre><code>
public static class MyFragment extends Fragment {
    CharSequence mLabel;

    //工厂方法，同时为fragment提供了arguments对象
    static MyFragment newInstance(CharSequence label) {
        MyFragment f = new MyFragment();
        Bundle b = new Bundle();
        b.putCharSequence("label", label);
        f.setArguments(b);
        return f;
    }

    //解析xml属性
    @Override public void onInflate(Activity activity, AttributeSet attrs,
            Bundle savedInstanceState) {
        super.onInflate(activity, attrs, savedInstanceState);
        TypedArray a = activity.obtainStyledAttributes(attrs,
                R.styleable.FragmentArguments);
        mLabel = a.getText(R.styleable.FragmentArguments_android_label);
        a.recycle();
    }

    //如果有arguments对象，就取其中的值（从xml中取到的被覆盖）
    @Override public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Bundle args = getArguments();
        if (args != null) {
            mLabel = args.getCharSequence("label", mLabel);
        }
    }

    //创建view
    @Override public View onCreateView(LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState) {
        View v = inflater.inflate(R.layout.hello_world, container, false);
        View tv = v.findViewById(R.id.text);
        ((TextView)tv).setText(mLabel != null ? mLabel : "(no label)");
        tv.setBackgroundDrawable(getResources().getDrawable(android.R.drawable.gallery_thumb));
        return v;
    }
}

</code></pre>

自定义的xml属性


    <declare-styleable name="FragmentArguments">
    	<attr name="android:label"/>
    </declare-styleable>

在Activity的布局文件中定义fragment（静态创建）


    <fragment class="com.example.android.apis.app.FragmentArguments$MyFragment"
    	android:id="@+id/embedded"
    	android:layout_width="0px" android:layout_height="wrap_content"
    	android:layout_weight="1"
    	android:label="@string/fragment_arguments_embedded" />

同样可以在代码中动态创建fragment

<pre><code>
@Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.fragment_arguments);
    if (savedInstanceState == null) {
        // First-time init; create fragment to embed in activity.
        FragmentTransaction ft = getFragmentManager().beginTransaction();
        Fragment newFragment = MyFragment.newInstance("From Arguments");
        ft.add(R.id.created, newFragment);
        ft.commit();
    }
}
</code></pre>



### 2. onAttach 

<pre><code>
public void onAttach (Context context)
</code></pre>
当Fragment与Activity关联之后调用一次。

此后，可使用Fragment.getActivity()获得与Fragment关联的窗口对象了。在此方法中，不可操作Fragment中的UI控件。


### 3. onCreate

<pre><code>
public void onCreate (Bundle savedInstanceState)
</code></pre>

用于读取保存的状态，获取或初始化一部分数据。

**注意：调用这个方法时，它所在的Activity还在创建过程中**

通常都会实现这一方法。

### 4. onCreateView

<pre><code>
public View onCreateView (LayoutInflater inflater, 
			  ViewGroup container, 
			  Bundle savedInstanceState)
</code></pre> 

<pre><code>
参数：
inflater	         LayoutInflater对象，inflater.inflate(……)
container	         如果非null，那么它就是fragment的UI需要绑定的父view。
                         它可用于创建合适的LayoutParams。
savedInstanceState	 如果非null，表明这个fragment是通过之前保存的状态重建的。
</code></pre>

创建Fragment的UI布局（一个View对象）。如果是无UI的fragment，返回null。
通常都会实现这一方法。

可以操纵Fragment内部的UI控件，但仍不能操纵Activity中的控件。


在创建View时，应调用：

<pre><code>
public View inflate (int resource, ViewGroup root, boolean attachToRoot)
</code></pre>

<pre><code>
resource：      需要加载的layout xml文件的id

root：          root就是onCreateView中的container，其作用分为两种情况：
		1. 如果attachToRoot设为true，那么root将作为将要产生的view的parent（这个比较少用）
		2. 如果attachToRoot设为false，那么root仅仅是用于为将要产生的view确定一组
		合适的LayoutParams（这种情况比较多见）
			    
attachToRoot：  将要产生的view是否要关联到root。如设为false，那么root仅仅用于为新创建的
                view确定一个合适的LayoutParams的子类。（通常设为false）
</code></pre>

**注意：动态创建的Fragment本身是无法布局的，它需要依附在一个“视图容器”（containter）中，Fragment永远是充满整个container的。所以通过设置container的布局就可以控制fragment的布局。**

例，在onCreateView中，可调用这样的代码
<pre><code>
container.setLayoutParams(new LayoutParams(...));
View v = inflater.inflate(layout_id, container, false);
return v;
</code></pre>

**在Fragment从回退栈返回时，也会调用此方法**

### 5. onViewCreated
<pre><code>
public void onViewCreated (View view, Bundle savedInstanceState)
</code></pre>

在onCreateView返回后立即被调用

### 6. onActivityCreated

<pre><code>
public void onActivityCreated (Bundle savedInstanceState)
</code></pre>

在Activity.onCreate之后被调用，表明Activity已经初始化完毕，Activity与Fragment已经绑定完毕。此后可以通过getActivity().findViewById方法来操纵Activity中的UI控件。

**在此方法中，可以调用setRetainInstance(boolean)。**


### 7. onViewStateRestored 
<pre><code>
public void onViewStateRestored (Bundle savedInstanceState)
</code></pre>
在所有保存的view的状态恢复后调用，表明fragment中所有view相关的状态都已恢复完毕。

可用于根据已保存的状态来进行view相关初始化，比如checkbox是否处于checked状态等。

此方法的调用时机在onActivityCreated之后，在onStart之前。


### 8. onStart
<pre><code>
public void onStart ()
</code></pre>
可见。在Activity.onState之后被调用


### 9. onResume 
<pre><code>
public void onResume ()
</code></pre>
可交互。在Activity.onResume之后被调用


### 10. onPause
<pre><code>
public void onPause ()
</code></pre>
不再交互。应保存持久化信息。在Activity.onPause之前被调用。


### 11. onStop
<pre><code>
public void onStop ()
</code></pre>
不可见。在Activity.onStop之前被调用

### 12. onDestroyView
<pre><code>
public void onDestroyView ()
</code></pre>

表明onCreateView创建的view与fragment分离（调用与否跟onCreateView是否返回null无关）。调用发生在onStop之后，onDestory之前。在内部，调用发生在view的state被保存之后，在view从它的parent移除之前。

**如果接下来fragment需要再次被显示，那么一个新的view将会被创建。**

可以在此处清理与view相关的资源。

**当fragment进入回退栈时，会调用此方法**


### 13. onDestroy
<pre><code>
public void onDestroy ()
</code></pre>

当fragment不再被使用时。可以做最终的清理工作。

### 14. onDetach
<pre><code>
public void onDetach ()
</code></pre>

fragment与Activity解除关联。在Activity.onDestory之前调用。

## 几个重要的函数

### 1. onSaveInstanceState

<pre><code>
public void onSaveInstanceState (Bundle outState)
</code></pre>

保存下来的状态可在onCreate(Bundle), onCreateView(LayoutInflater, ViewGroup, Bundle), and onActivityCreated(Bundle)中取到。

此方法会在onDestroy之前的任何时间点被调用。

**有很多情况下，fragment会被剥离（比如放入回退栈），但是除非它所在的Activity需要保存状态，否则onSaveInstance不会被调用。也就是说，把一个Fragment放入回退栈，不会造成onSaveInstanceState被调用。**

### 2. getView

public View getView ()

Get the root view for the fragment's layout (the one returned by onCreateView(LayoutInflater, ViewGroup, Bundle)), if provided.


### 3. setRetainInstance


## 创建方式

### 1. 静态创建

通过xml文件创建。一旦系统加载了含有该fragment的布局文件，此fragment的实例就被创建。

### 2. 动态创建

在代码中创建。分为以下几步：

- 获取FragmentManager。

<pre><code>
fmgr = getFragmentManager()
</code></pre>

- 开启事务。

<pre><code>
transaction = fmgr.beginTransaction()
</code></pre>

- 创建Fragment

<pre><code>
f = new XXXFragment(args...)

or

f = XXXFragment.newInstance(args...)
</code></pre>

- 将Fragment加入事务(add/remove/replace/attach/detach/show/hide)

<pre><code>
transaction.add(containerId, f)
</code></pre>

- 提交事务

<pre><code>
transaction.commit()
</code></pre>

## 交互

Fragment之间是不应该发生直接交互的，应该通过与它们关联的Activity来进行。

基本方法就是通过回调，具体来讲：

（1）定义一个interface（可以在Fragment内部定义内部interface，也可以直接定义成普通的interface）

（2）与Fragment关联的Activity实现这个接口。在Fragment.onAttach中可以通过getActivity()得到此Activity，并将其强转为interface的类型。后面就可以调用这个interface的方法了。

例子：

（1）定义接口
<pre><code>
public class HeadlinesFragment extends ListFragment {
    OnHeadlineSelectedListener mCallback;
    
    //定义一个内部接口，Activity必须实现之
    public interface OnHeadlineSelectedListener {
        public void onArticleSelected(int position);
    }


    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);
        
        //将Activity强转为内部接口的对象
        try {
            mCallback = (OnHeadlineSelectedListener) activity;
        } catch (ClassCastException e) {
            throw new ClassCastException(activity.toString()
                    + " must implement OnHeadlineSelectedListener");
        }
    }

	@Override
    public void onListItemClick(ListView l, View v, int position, long id) {
        
		// 向Activity传递信息
        mCallback.onArticleSelected(position);
    }    

    ...
}
</code></pre>

接下来，就可以利用mCallback调用接口的方法（比如onAriticleSelected)了。

（2）实现接口

Activity一方面可以处理来自fragment的信息（通过回调），另一方面可以通过FragmentManager.findFragmentById来取得fragment对象，并调用它的public方法。

当信息来自Fragment A，而通过findFragmentById获得了Fragment B，就可以实现A到B的信息传递了。

<pre><code>

public static class MainActivity extends Activity
        implements HeadlinesFragment.OnHeadlineSelectedListener{
    ...
    
    public void onArticleSelected(int position) {

        // 参数position是来自于HeadlinesFragment的信息
        // 通过findFragmentById取得ArticleFragment的对象，并将position传递给它
        ArticleFragment articleFrag = (ArticleFragment)
                getSupportFragmentManager().findFragmentById(R.id.article_fragment);
        if (articleFrag != null) {

            // 如果存在，说明布局中可以同时存在两个fragment，那么直接更新内容就可以了
            articleFrag.updateArticleView(position);
        } else {

            // 否则，说明布局中只有一个fragment，就要创建新fragment，并替换之           
            ArticleFragment newFragment = new ArticleFragment();
            Bundle args = new Bundle();
            args.putInt(ArticleFragment.ARG_POSITION, position);
            newFragment.setArguments(args);
        
            FragmentTransaction transaction = getSupportFragmentManager

            //替换并加入回退栈
            transaction.replace(R.id.fragment_container, newFragment);
            transaction.addToBackStack(null);
            
            //提交
            transaction.commit();
        }
    }
}

</code></pre>


## Arguments

Activity可以通过setArguments向Fragment传递参数，但这**必须发生在onAttach之前**。这个Arguments（一个Bundle对象）将在fragment的整个生命周期期间存在。如果未设置，getArguments将返回null。

<pre><code>
public final Bundle getArguments()

public void setArguments (Bundle args)
</code></pre>

**注意：
setArguments只能在Fragment与Activity关联（attach）之前调用，也就是说必须在构建fragment之后立即调用**。
比较常见的有两个地方：

- 在onInflate方法中
- 在工厂方法中，比如XXXFragment.newInstance

与setArguments不同，getArguments可以在任何时间点调用。

## 持久化

- 发生configChanges时，会导致onSaveInstanceState被调用。
- fragment放入回退栈时，不会导致onSaveInstanceState被调用。那么当从回退栈中返回时，会丢失一些信息（比如成员变量）。因此，在进入回退栈之前，需要保存信息。在哪里保存？在onDestroyView中。onDestroyView中没有bundle怎么办？用fragment的Arguments对象。这个对象怎么得到？用getArguments方法。

- 假如当前有两个Fragment（A和B）被放入回退栈，A在上，B在下，A目前处于显示状态。此时旋转屏幕，出发configChanges，此时A和B的onSaveInstanceState都会被调用。**如果在B的onSaveInstanceState方法中涉及保存其View的状态，会抛出异常！！！**这是因为B在加入回退栈后，处于了不显示的状态，它的onDestroyView被调用，它的view已经不存在了，在onSaveInstanceState中去访问其View对象必然抛出异常。**必须要通过getView来检查！！**

**注意：在恢复和保存View状态时，应先通过getView确保其不为null**

<pre><code>
public class StatedFragment extends Fragment {

    Bundle savedState;

    public StatedFragment() {
        super();

        //确保Arguments对象存在
        if(getArguments() == null){
			setArguments(new Bundle());
        }
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        // 从Arguments恢复状态
        if (!restoreStateFromArguments()) {
            // First Time, Initialize something here
            onFirstTimeLaunched();
        }
    }

    protected void onFirstTimeLaunched() {

    }

    @Override
    public void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        //保存状态到Arguments中
        saveStateToArguments();
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        // 保存状态到Arguments中
        saveStateToArguments();
    }

    
    //保存状态到Arguments中
    private void saveStateToArguments() {

        //检查view是否还存在
        if (getView() != null)
            //建立一个状态bundle
            savedState = saveState();
        if (savedState != null) {
            Bundle b = getArguments();
            //将状态bundle存入Arguments中
            b.putBundle("internalSavedViewState8954201239547", savedState);
        }
    }

    
    //恢复
    private boolean restoreStateFromArguments() {
        Bundle b = getArguments();
        //从Arguments中取出状态bundle
        savedState = b.getBundle("internalSavedViewState8954201239547");
        if (savedState != null) {
            restoreState();
            return true;
        }
        return false;
    }


    //将状态bundle中的内容恢复出来
    private void restoreState() {
        if (savedState != null) {
            // For Example
            //tv1.setText(savedState.getString("text"));
            onRestoreState(savedState);
        }
    }

    protected void onRestoreState(Bundle savedInstanceState) {

    }

    //获取状态bundle
    private Bundle saveState() {
        Bundle state = new Bundle();
        // For Example
        //state.putString("text", tv1.getText().toString());
        onSaveState(state);
        return state;
    }

    protected void onSaveState(Bundle outState) {

    }
}

</code></pre>



## FragmentManager

### 获取FragmentManager

<pre><code>
Activity.getFragmentManager()

Fragment.getFragmentManager()
</code></pre>

### 查找Fragment

<pre><code>
FragmentManager.findFragmentById (int id)

FragmentManager.findFragmentByTag (String tag)
</code></pre>

### 存取Fragment

<pre><code>
FragmentManager.getFragment (Bundle bundle, String key)
FragmentManager.putFragment (Bundle bundle, String key, Fragment fragment)
</code></pre>

这两个方法在Activity中调用。将Fragment作为普通对象存取。其中的Bundle可以分别为Activity.onCreate和Activity.onSaveInstanceState的参数。

## FragmentTransaction

### 开启事务
得到FragmentTransaction对象。
<pre><code>
FragmentManager.beginTransaction ()
</code></pre>

注意：创建和提交事务必须在Activity保存状态之前（Activity.onSaveInstanceState之前）

### 添加

- 无tag
<pre><code>
FragmentTransaction.add (int containerViewId, Fragment fragment)
</code></pre>

- 无UI
<pre><code>
FragmentTransaction.add (Fragment fragment, String tag)
</code></pre>

- tag可用于findFragmentByTag
<pre><code>
FragmentTransaction.add (int containerViewId, Fragment fragment, String tag)
</code></pre>

### 移除

<pre><code>
FragmentTransaction.remove (Fragment fragment)
</code></pre>

### 替换

<pre><code>
FragmentTransaction.replace (int containerViewId, Fragment fragment, String tag)

FragmentTransaction.replace (int containerViewId, Fragment fragment)
</code></pre>

### 绑定与解绑定

<pre><code>
FragmentTransaction.attach (Fragment fragment)
</code></pre>

将之前解绑的fragment再次绑定。
<pre><code>
FragmentTransaction.detach (Fragment fragment)
</code></pre>

解绑定只是将Fragment从它所在的Activity移走，后面可以通过attach再次绑定。detach之后，它的状态仍然被FragmentManager管理。

**注意detach与remove的区别与联系：两者都将fragment从Activity移走，但detach可再次attach，fragment能够被再次使用，remove则将fragment恢复至初始状态，不可再用。**

### 显示与隐藏

<pre><code>
FragmentTransaction.show (Fragment fragment)
FragmentTransaction.hide (Fragment fragment)
</code></pre>

### 提交事务

<pre><code>

FragmentTransaction.commit ()
</code></pre>

提交并非立即生效，系统会进行调度。
注意：创建和提交事务必须在Activity保存状态之前（Activity.onSaveInstanceState之前）

如果希望之前commit的事务能够立即生效，可以调用（必须在主线程调用）：
<pre><code>
FragmentManager.executePendingTransactions ()
</code></pre>

## Fragment与回退栈



## 重要的Fragment

### ListFragment

### DialogFragment

### PreferenceFragment

一个guide 5个training