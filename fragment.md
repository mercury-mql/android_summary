#Fragment

***

## 生命周期

Fragment必须内嵌在Activity中，并且Fragment的生命周期受到它所在的Activity的影响。

Fragment可以作为Activity UI的一部分，也可以设置为无UI的Fragment，从而作为Activity的不可见工作者（invisible worker）。



The core series of lifecycle methods that are called to bring a fragment up to resumed state (interacting with the user) are:










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


### 7. onViewStateRestored(Bundle) 
tells the fragment that all of the saved state of its view hierarchy has been restored.
Called when all saved state has been restored into the view hierarchy of the fragment. This can be used to do initialization based on saved state that you are letting the view hierarchy track itself, such as whether check box widgets are currently checked. This is called after onActivityCreated(Bundle) and before onStart().


### 8. onStart() 
makes the fragment visible to the user (based on its containing activity being started).


### 9. onResume() 

<pre><code>
public void onResume ()
</code></pre>

makes the fragment begin interacting with the user (based on its containing activity being resumed).



Added in API level 11
Called when the fragment is visible to the user and actively running. This is generally tied to Activity.onResume of the containing Activity's lifecycle.

As a fragment is no longer being used, it goes through a reverse series of callbacks:

### 9. onPause()
The system calls this method as the first indication that the user is leaving the fragment (though it does not always mean the fragment is being destroyed). This is usually where you should commit any changes that should be persisted beyond the current user session (because the user might not come back).

Called when the Fragment is no longer resumed. This is generally tied to Activity.onPause of the containing Activity's lifecycle.




onPause() fragment is no longer interacting with the user either because its activity is being paused or a fragment operation is modifying it in the activity.


### 10. onStop() 
fragment is no longer visible to the user either because its activity is being stopped or a fragment operation is modifying it in the activity.

### 11. public void onDestroyView ()

Added in API level 11
Called when the view previously created by onCreateView(LayoutInflater, ViewGroup, Bundle) has been detached from the fragment. The next time the fragment needs to be displayed, a new view will be created. This is called after onStop() and before onDestroy(). It is called regardless of whether onCreateView(LayoutInflater, ViewGroup, Bundle) returned a non-null view. Internally it is called after the view's state has been saved but before it has been removed from its parent.
onDestroyView() allows the fragment to clean up resources associated with its View.

### 12. public void onDestroy ()

Added in API level 11
Called when the fragment is no longer in use. This is called after onStop() and before onDetach().

onDestroy() called to do final cleanup of the fragment's state.

### public void onDetach ()

Added in API level 11
Called when the fragment is no longer attached to its activity. This is called after onDestroy().

onDetach() called immediately prior to the fragment no longer being associated with its activity.

## 创建Fragment

## 几个重要的函数


## 加入Activity的方式

### 1. 静态创建

### 2. 动态创建

## 有Activity的交互

## 持久化

## 事务

## 重要的Fragment

### ListFragment

### DialogFragment

### PreferenceFragment

一个guide 5个training