NavigationView集成自FrameLayout，就是一样简单的容器，其内部会使用一个listView来显示列表项。
```
	public class NavigationView extends ScrimInsetsFrameLayout
```
看看NavigationView的几个private field
```
	private final MenuBuilder mMenu;
    private final NavigationMenuPresenter mPresenter;
    private NavigationView.OnNavigationItemSelectedListener mListener;
```
MenuBuilder其实是继承自Menu，可以直接从menu资源中加载我们自定义的菜单，用来显示在listView中。可以这么理解，menu这里充当了listView的model，这样我们就可以方便的使用xml来自定义listView中显示的数据和样式，listView的adapter会从menu中读取要显示的数据。下面的代码就是来自NavigationView构造函数，使用LayoutInflator来将xml中的数据读取到menu中。
```
	this.inflateMenu(a.getResourceId(styleable.NavigationView_menu, 0));

	this.getMenuInflater().inflate(resId, this.mMenu);
```

NavigationMenuPresenter，看到presenter是不是很熟悉，没错，就是MVP，看来MVP在Android中势不可挡啊，Google已经开始使用了。
在这里，NavigationView将创建子view的任务完全委托给了presenter。
```
	this.mPresenter = new NavigationMenuPresenter();
	this.mPresenter.setId(1);
	this.mPresenter.initForMenu(context, this.mMenu);
	this.mPresenter.setItemIconTintList(itemIconTint);
	this.mPresenter.setItemTextColor(itemTextColor);
	this.mPresenter.setItemBackground(itemBackground);
	this.mMenu.addMenuPresenter(this.mPresenter);
	this.addView((View)this.mPresenter.getMenuView(this));
```
NavigationMenuPresenter里面所做的事情就是给mMenu这个listView对象创建一个adapter。
```
	public MenuView getMenuView(ViewGroup root) {
        if(this.mMenuView == null) {
            this.mMenuView = (NavigationMenuView)this.mLayoutInflater.inflate(layout.design_navigation_menu, root, false);
            if(this.mAdapter == null) {
                this.mAdapter = new NavigationMenuPresenter.NavigationMenuAdapter();
            }

            this.mHeader = (LinearLayout)this.mLayoutInflater.inflate(layout.design_navigation_item_header, this.mMenuView, false);
            this.mMenuView.addHeaderView(this.mHeader);
            this.mMenuView.setAdapter(this.mAdapter);
            this.mMenuView.setOnItemClickListener(this);
        }

        return this.mMenuView;
    }
```

NavigationView整体分析下来感觉就是这也忒简单了吧。其实仔细想想还是有两个点很值得我学习的。
1.NavigationView其实和我以前自己写的思路也是差不多，都是使用listView来显示。自己也尝试过做成一个通用的模式，但是每次都要在代码里去生成数据，也尝试使用过xml定义，但是使用普通的string资源很难去定义类似subMenu这种。
Google这里就给出了一个很好的思路，既然Android已经提供了Menu这个类，并且Menu可以使用xml来定义，xml中还可以定义subMenu，为啥不使用Menu来定义我们的侧边栏呢？
2.MVP的确挺适合Android的，使用MVP之后，可以使view变得简单很多。