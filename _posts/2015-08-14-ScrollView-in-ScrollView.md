---
published: true
title: Android ScrollView嵌套ListView、GridView的办法及推荐作法
layout: post
author: 代盼华
category: 技术
tags: 
- ScrollView
- 嵌套
- ListView
---


#问题描述#
在Android开发中，我们经常会碰到ScrollView嵌套可滚动View的需求，例如ScrollView嵌套ListView。但是如果不加处理，直接这么做的话，会出现被嵌套的View高度不正常的问题。

#问题出现的原因#
其实谷歌是不建议我们在ScrollView里嵌套ListView的，一来不符合开发规范；二来，这么做将会导致ListView的item复用机制无效，会导致内存的问题。
至于出现被嵌套View高度不一致的原因，网上可以搜到很多结果。

#解决办法#
解决办法无非有以下两种：

1. 获取每个Item高度，计算累加后给被嵌套的View重置高度：

{% highlight java %}

    public static void setListViewHeightBasedOnChildren(ListView listView) {

		ListAdapter listAdapter = listView.getAdapter();		
		if (listAdapter == null) {
			return;	
		}
		
		int totalHeight = 0;
		for (int i = 0; i < listAdapter.getCount(); i++) {
			View listItem = listAdapter.getView(i, null, listView);			
			listItem.measure(0, 0);			
			totalHeight += listItem.getMeasuredHeight();		
		}
		
		ViewGroup.LayoutParams params = listView.getLayoutParams();
		
		params.height = totalHeight
		+ (listView.getDividerHeight() * (listAdapter.getCount() - 1));
		
		listView.setLayoutParams(params);	
	}

{% endhighlight java %}

2. 重写子View，通过"欺骗父控件"，告诉父控件“我需要很大很大的空间”,即重写子View的如下方法：


{% highlight java %}

	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
			int expandSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2,
			MeasureSpec.AT_MOST);
			super.onMeasure(widthMeasureSpec, expandSpec);
 	}


{% endhighlight java %}

通过这种方式，可能会出现页面加载时，ScrollView向下发生了滚动，此时调用`ScrollView.smoothScrollTo(0, 0);`方法就好了

#哪种解决办法更好？#
如果数据量比较大的话(我这里套的是gridView，就6个item，计算高度却用了800ms，应该与item的ui复杂度有关)，通过第一种方式循环计算每个item的高度会比较耗时，而采用第二种————重写View的方式，就不用for循环计算高度了。


