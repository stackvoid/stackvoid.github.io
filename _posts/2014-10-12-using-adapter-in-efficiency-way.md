---
layout: post
title: Android数据适配器(Adapter)优化：使用高效的ViewHolder
categories: [Android]
tags: [优化]
---

在使用Listview或GridView的时候，往往需要自定义数据适配器，一般都要覆写getView()，在该方法中有一个convertView参数，该参数就是用来加载数据时的View。 

###初学者简单但低效的方式

{%highlight java linenos%}
public View getView(int position, View convertView, ViewGroup parent) {
	
	View item= inflater.inflate(R.layout.good_list_item, null, false);
		 
	ImageView img = (ImageView) item.findViewById(R.id.img);
	TextView price = (TextView) item.findViewById(R.id.price);
	img.setImageResource(R.drawable.ic_launcher);
	price.setText("$"+list.get(position).price);
			
	return item;
}
{%endhighlight%}

每次加载view，都要重新建立很多view对象，如果某条listview中有一万条数据，这种加载方式就歇菜了。

###利用convertView

利用Android的Recycler机制，利用convertView来重新回收View，效率有了本质提高。View的每次创建是比较耗时的，因此对于getview方法传入的convertView应充分利用 != null的判断 。

{%highlight java linenos%}
public View getView(int position, View convertView, ViewGroup parent) {

		if(convertView==null){
			convertView = inflater.inflate(R.layout.good_list_item, null, false);
		}
		TextView tv_price = (TextView)convertView.findViewById(R.id.price)
		ImageView iv = (ImageView)convertView.findViewByID(R.id.img);
		
		return convertView;
	}

{%endhighlight%}

###使用ViewHolder

ViewHolder将需要缓存的view封装好，convertView的setTag才是将这些缓存起来供下次调用。
当你的listview里布局多样化的时候 viewholder的作用体现明显，效率再一次提高。
View的findViewById()方法也是比较耗时的，因此需要考虑只调用一次，之后就用View.getTag()方法来获得ViewHolder对象。 

{%highlight java linenos%}
class ViewHolder{
		ImageView img;
		TextView price;
	}
public View getView(int position, View convertView, ViewGroup parent) {
		ViewHolder holder = new ViewHolder();
		if(convertView==null){
			convertView = inflater.inflate(R.layout.good_list_item, null, false);
			holder.img = (ImageView) convertView.findViewById(R.id.img);
			holder.price = (TextView) convertView.findViewById(R.id.price);
			convertView.setTag(holder);  
		}else{
			holder = (ViewHolder) convertView.getTag();
		}
		//设置holder
		holder.img.setImageResource(R.drawable.ic_launcher);
		holder.price.setText("$"+list.get(position).price);
			
		return convertView;
	}

{%endhighlight%}

###优雅的使用ViewHolder

使用ViewHolder时，每次一遍一遍的findViewById，一遍一遍在ViewHolder里面添加View的定义，view一多，是不是感觉烦爆了，[base-adapter-helper](https://github.com/JoanZapata/base-adapter-helper)这个类库似乎完美的解决了这个问题。

其设计思想是使用 SparseArray<View>来存储view的引用，代替了原本的ViewHolder，不用声明一大堆View，简洁明了。

我也自己动手写了一个简单版的ViewHolder。
{%highlight java linenos%}
public class ViewHolder{
 
    private final SparseArray<View> views;
    private View convertView;

     private ViewHolder(View convertView){
        this.views = new SparseArray<View>();
        this.convertView = convertView;
        convertView.setTag(this);
    }

    public static ViewHolder get(View convertView){
        if (convertView == null) {
            return new ViewHolder(convertView);
        }
        ViewHolder existedHolder = (ViewHolder) convertView.getTag();
        return existedHolder;
    }
 
    public <T extends View> T getView(int viewId) {
        View view = views.get(viewId);
        if (view == null) {
            view = convertView.findViewById(viewId);
            views.put(viewId, view);
        }
        return (T) view;
    }
}
{%endhighlight%}

使用的话就超级简单和简洁了：

{%highlight java linenos%}
   public View getView(int position, View convertView, ViewGroup parent) {
        if (convertView == null) {
            convertView = LayoutInflater.from(context)
                    .inflate(R.layout.good_list_item, null, false);
        }
 
        ViewHolder mViewHolder = ViewHolder.get(convertView);
        TextView price = mViewHolder.getView(R.id.price);
        //...其他getView
 
        return convertView;
    }

{%endhighlight%}



[类似这种情况不要使用ViewHolder](http://blog.xebia.com/2013/07/22/viewholder-considered-harmful/#comment-365736)























