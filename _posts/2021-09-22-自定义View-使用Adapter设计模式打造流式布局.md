---
title: 使用适配器设计模式打造流式布局
tags: 自定义View
permalink: android-source/dc-view-3
key: android-source-dc-view-3
sidebar:
  nav: android-source
---

### 概述
最近在写公司项目需求，有一个流式布局FlowLayout需要进行修改，这个一般用于显示标签信息，看了一下之前的代码，感觉可拓展性太差，要实现这次的效果，有点麻烦，索性自己打造一个流式布局，这次决定像ListView，RecycleView一样，采用Adapter设计模式进行实现，这样具体对某个子View进行修改的话，可以直接在Adapter中进行修改。

<!--more-->

### 效果

![view_03_01.webp](https://raw.githubusercontent.com/QingDian-Fan/ImageRepository/master/images/UdJuM3XFhs6zyn9-20231013.webp)
### 分析
1.继承自谁？我们可以看到内部包含许多View，因此这是一个自定义ViewGroup

2.测量 onMeasure(),这里我们需要知道FlowLayout的高度可以随便给，但是宽度不可以是wrap_content，因此我们需要在对wrap_conent时抛出异常提醒,具体怎么判断宽度是否为wrap_content? 这里需要根据测量模式进行处理，简单可以理解为：

2.1 EXACTLY：表示设置了精确值，一般当设置其宽、高为dp或是match_parent时，会将其测量模式设置为EXACTLY；
2.2 AT_MOST：一般当设置其宽、高为wrap_content时，会将其测量设置为AT_MOST；
2.3 UNSPECIFIED：表示子布局想要多大就多大，此种模式比较少见。一般会在系统的View中，例如ScrollView...

3.摆放 onLayout：这里我们需要对子View按照顺序进行换行或摆放，定位
### 实现
1.首先需要处理的是onMeasure()方法

1.1 需要处理什么时候换行？

当前已摆放View的宽度+即将要摆放View的宽度>该FlowLayout的宽度+PaddingLeft+PaddingRight

1.2 高度计算

FlowLayout的高度 = 所有行的最大高度+PaddingTop+PaddingBottom

1.3 当View设置为GONE则不进行测量

注意⚠️：这里我们设置了两个List集合，一个是我们所有的集合按照我们每一行的View进行储存，为了避免代码臃肿，在onlayout在写一遍，这个集合我们在这里进行储存，我们在摆放时需要使用

下面看下具体的实现代码


```java
private List<List<View>> mChildViews = new ArrayList<>();
private ArrayList<View> childViews = new ArrayList<>();//每一行的View
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    //判断用户是否设置宽度为wrap_content，如果是wrap_content则抛出异常
    if (MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.AT_MOST) {
        throw new RuntimeException("FlowLayout does not allow setting layout_width to wrap_content");
    }
    mCurrentLines = 0;
    //获取View个数
    int childCount = getChildCount();
    //获取宽度
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    //计算高度
    int heightSize = getPaddingTop() + getPaddingBottom();
    int lineWidth = 0;
    int maxLineHeight = 0;
    mChildViews.clear();
    childViews.clear();
    //循环所有的View计算高度  注意处理Gone,padding margin
    for (int i = 0; i < childCount; i++) {
        //获取子View
        View mChildView = getChildAt(i);
        if (mChildView.getVisibility() == GONE) continue;
        //这句话执行完毕之后  就可以获取子View的宽高了 因为这句话会调用子View 的measure方法
        measureChild(mChildView, widthMeasureSpec, heightMeasureSpec);
        //获取layoutParams  计算最大高度
        MarginLayoutParams layoutParams = (MarginLayoutParams) mChildView.getLayoutParams();
        maxLineHeight = Math.max(maxLineHeight, mChildView.getMeasuredHeight() + layoutParams.topMargin + layoutParams.bottomMargin);
        int mChildWidth = mChildView.getMeasuredWidth() + layoutParams.leftMargin + layoutParams.rightMargin;

        if (lineWidth + mChildWidth > widthSize - getPaddingLeft() - getPaddingRight()) {
            // 需要换行
            heightSize += maxLineHeight;
            maxLineHeight = 0;
            lineWidth = mChildWidth;
            mChildViews.add(childViews);
            childViews = new ArrayList<>();
            mCurrentLines++;
            if (lines != 0 && mCurrentLines == lines) break;
        } else {
            //未满一行不需要换行
            lineWidth += mChildWidth;
        }

        childViews.add(mChildView);
        if (i == (childCount - 1)) {
            heightSize += maxLineHeight;
            mChildViews.add(childViews);
        }
    }
    //设置自己的宽高
    setMeasuredDimension(widthSize, heightSize);
}
```

2. onLayout摆放方法，这里我们需要注意的是处理好子View的Margin和FlowLayout的Padding，这里拿到我们在onMeasure中赋值的集合，在这里进行循环摆放，

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int left, top = this.getPaddingTop(), right, bottom;
    int maxLineHeight = 0;
    //循环所有的
    for (List<View> mViews : mChildViews) {
        //新的一行开始
        left = getPaddingLeft();
        for (View mView : mViews) {
            MarginLayoutParams layoutParams = (MarginLayoutParams) mView.getLayoutParams();
            maxLineHeight = Math.max(maxLineHeight, mView.getMeasuredHeight() + layoutParams.topMargin + layoutParams.bottomMargin);
            left += layoutParams.leftMargin;
            right = left + mView.getMeasuredWidth();
            bottom = top + layoutParams.topMargin + mView.getMeasuredHeight();
            mView.layout(left, top, right, bottom);
            left += mView.getMeasuredWidth() + layoutParams.rightMargin;
        }
        top += maxLineHeight;
    }
}
```

到这里，一个流式布局大致完成，不过这样的流式布局，使用及后续的修改比较麻烦，再者就是在添加数据时，一个数据的多样性，想String[],List<String>都比较麻烦，下面我们仿照ListView，RecycleView一样使用Adapter设计模式来解决这些耦合度问题，如果数据变化的话，我们也按照ListView，RecycleView一样采用观察者模式，进行刷新.下面就来看下具体的实现吧

1. 首先我们需要定义一个抽象的类，我们关心的条目个数，及View显示需要定义为抽象方法，至于其他的公共使用的，我们在该类中实现，

```java
public abstract class FlowLayoutAdapter {
    private final DataSetObservable mDataSetObservable = new DataSetObservable();
    // 子view的集合
    private final ArrayList<View> views = new ArrayList<>();

    public void registerDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.registerObserver(observer);
    }

    public void unregisterDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.unregisterObserver(observer);
    }

 
    public void notifyDataSetChanged() {
        mDataSetObservable.notifyChanged();
    }

    // 得到条目数
    public abstract int getItemCount();

    // 根据位置得到子View布局
    public abstract View getItemView(int position, ViewGroup parent);

    // 将子view布局添加到总的list里面
    public void addViewToList(ViewGroup parent) {
        views.clear();
        int counts = getItemCount();
        if (counts == 0) return;

        for (int i = 0; i < counts; i++) {
            views.add(getItemView(i, parent));
        }
    }

    //得到列表里面的所有子view
    public ArrayList<View> getViewList() {
        return views;
    }
}
```

2.在FlowLayout中设置setAdapter()方法添加View

```java
public void setAdapter(FlowLayoutAdapter adapter) {
    if (mAdapter != null && mDataSetObserver != null) {
        //注销观察者
        mAdapter.unregisterDataSetObserver(mDataSetObserver);
        mAdapter = null;
    }

    if (adapter == null) throw new NullPointerException("adapter does not allow is null");
    

    this.mAdapter = adapter;
    //刷新数据
    mDataSetObserver = new DataSetObserver() {
        @Override
        public void onChanged() {
            resetLayout();
        }
    };
    //注册观察者
    mAdapter.registerDataSetObserver(mDataSetObserver);
    resetLayout();
}

protected final void resetLayout() {
    this.removeAllViews();
    int counts = mAdapter.getItemCount();
    mAdapter.addViewToList(this);
    ArrayList<View> views = mAdapter.getViewList();
    for (int i = 0; i < counts; i++) {
        this.addView(views.get(i));
    }
}
```

3. 下面我们看一下具体使用实例
3.1 在Activity中：

```java
class FlowLayoutActivity : AppCompatActivity() {
    companion object {
        fun start(mContext: Context, title: String) {
            val intent = Intent()
            intent.setClass(mContext, FlowLayoutActivity::class.java)
            intent.putExtra("title", title)
            mContext.startActivity(intent)
        }
    }

    private val datas: ArrayList<String> = arrayListOf(...)
    
    private val mFlowLayout: FlowLayout by lazy { findViewById(R.id.flowLayout) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_flow_layout)
        mFlowLayout.setAdapter(DataFlowAdapter(this@FlowLayoutActivity, datas))
    }
}
```

3.2 在Adapter中

```java
public class DataFlowAdapter extends FlowLayoutAdapter {
    private Context mContext;
    private List<String> dataList;

    public DataFlowAdapter(Context mContext, List<String> dataList) {
        this.mContext = mContext;
        this.dataList = dataList;
    }

    @Override
    public int getItemCount() {
        return dataList == null ? 0 : dataList.size();
    }

    @Override
    public View getItemView(int position, ViewGroup parent) {
        AppCompatTextView mView = (AppCompatTextView) LayoutInflater.from(mContext).inflate(R.layout.item_flow_view, parent, false);
        mView.setText(dataList.get(position));
        return mView;
    }
}
```

至此，整个流式布局到这里也就完成了

代码地址：[https://gitee.com/QingDian_Fan/FlowLayoutProject](https://gitee.com/QingDian_Fan/FlowLayoutProject)







