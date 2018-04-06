## 从 fragment 中启动 activity
从 fragment 中启动 activity ，和从 activity 中启动 activity 是一样的，如图。
![image](https://note.youdao.com/yws/api/personal/file/4393CBB56FD642D78220632C566E3835?method=download&shareKey=98b58dac380e4dc70a4f75293f8ac78f)

需要注意的小点是，newIntent 方法是写在 CrimeActivity 中的。在CrimeFragment 中调用  getActivity().getIntent().getSerializableExtra(...) 得到值。

**CrimeActivity.class**
```
public static Intent newIntent(Context packageContext, UUID crimeId){
    Intent intent = new Intent(packageContext,CrimeActivity.class);
    intent.putExtra(EXTRA_CRIME_ID,crimeId);
    return intent;
}
```
[GitHub地址](https://github.com/YueJZJM/argument01)

那么为什么又要用 argument 呢？因为这破环了 CrimeFragment 的封装，CrimeActivity 无法用于其他的 Activity 了。比较好的做法是把 crime ID 存放到 CrimeFragment  的某个地方，而不是保存到 CrimeActivity 的私有空间中。这样就无需依赖 CrimeActivity  的私有空间了。这样就不需要依赖 CrimeActivity 的 intent 内的 extra，CrimeFragment 就能获取自己所需要的数据。属于fragment 的“某个地方”就是 argument bundle。

要附加 argument bundle 给 fragment。需要调用 Fragment.setArguments(Bundle) 方法。而且必须在 fragment 创建后，添加给 activity 前。为了满足以上的要求，添加 newInstance() 方法，完成 fragment 实例和 Bundle 的创建。

那么如何使用呢？我们从头来讲。

首先还是当然还是需要启动 CrimeActivity。在 CrimeListFragment 的 CrimeHolder 中启动。

**CrimeListFragment.class**
```
@Override
public void onClick(View view) {
    Intent intent = CrimeActivity.newIntent(getActivity(),mCrime.getId());
    startActivity(intent);
}
```
在 CrimeActivity 中得到 id 并传给 CrimeFragment。

**CrimeActivity.class**
```
 @Override
protected Fragment createFragment() {
    UUID crimeId = (UUID) getIntent().getSerializableExtra(EXTRA_CRIME_ID);
    return  CrimeFragment.newInstance(crimeId);
}
```
CrimeFragment 存放 id 到 argument并创建 fragment。

**CrimeFragmen.classt**
```
private static final String ARG_CRIME_ID = "crime_id";
public static CrimeFragment newInstance (UUID crimeId){
    Bundle args = new Bundle();
    args.putSerializable(ARG_CRIME_ID,crimeId);

    CrimeFragment fragment = new CrimeFragment();
    fragment.setArguments(args);
    return fragment;
}
```
在 onCreat(Bundle savedInstanceState) 中得到

**CrimeFragment.class**
```
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    UUID crimeId = (UUID) getArguments().getSerializable(ARG_CRIME_ID);
    mCrime = CrimeLab.get(getActivity()).getCrime(crimeId);
}
```
现在数据可以正常显示了，那么我想点击某个列表项，然后修改它的值，返回列表项后，可以直接看到修改后的值，该怎么操作呢？

其实我们修改后，数据已经保存到模型层了，我们只需要刷新 RecycleView 就可以了。调用 notifyDataSetChanged() 就行了。这里需要注意的是，要在 onResume() 方法中写。

**CrimeListFragment.class**
```
 @Override
public void onResume() {
    super.onResume();
    updateUI();
}

private void updateUI() {
    CrimeLab crimeLab = CrimeLab.get(getActivity());
    List<Crime> crimes = crimeLab.getCrimes();
    if (mAdapter==null){
        mAdapter = new CrimeAdapter(crimes);
        mCrimeRecyclerView.setAdapter(mAdapter);
    }else {
        mAdapter.notifyDataSetChanged();
    }
}
```

现在又有一个新的问题，notifyDataSetChanged() 是更新所有的列表项，但是我们每次只修改一个，怎么优化这一步操作呢？可以调用 RecycleView.Adapter 的 notifyItemChanged(int) 方法。

增加一个变量：

**CrimeListFragment.class**
```
//保存位置变量
private int mPosition;
...
//得到位置
@Override
public void onClick(View view) {
    Intent intent = CrimeActivity.newIntent(getActivity(),mCrime.getId());
    mPosition = this.getAdapterPosition();
    startActivity(intent);
}

...
//更新某个位置的数据
private void updateUI() {
    CrimeLab crimeLab = CrimeLab.get(getActivity());
    List<Crime> crimes = crimeLab.getCrimes();
    if (mAdapter==null){
        mAdapter = new CrimeAdapter(crimes);
        mCrimeRecyclerView.setAdapter(mAdapter);
    }else {
       // mAdapter.notifyDataSetChanged();
        mAdapter.notifyItemChanged(mPosition);
    }
}

```
[GitHub地址](https://github.com/YueJZJM/argument02)
