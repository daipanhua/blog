---
published: true
title: Android 中异步filter的使用
layout: post
author: 代盼华
category: 技术
tags: 
- 
---


#filter的作用#

今天看`android.widget.AutoCompleteTextView`的源码，想了解源码里是如何实现字符串自动匹配的，主要是学习下源码的思路，于是发现了`android.widget.Filter`这个类，这个类的原理和使用与`android.os.AsyncTask`很类似，通过Thread和Handler组合实现异步任务。
而在AutoCompleteTextView中，字符串的匹配通过异步任务的方式，计算以xxx开头的字符串并将其添加到一个list中，最后再将list输出到主线程，供ArrayAdapter调用并刷新数据集，通过这种异步线程的方式，实现良好的用户体验。

#源码解读#
##一、先看下AutoCompleteTextView的setAdapter方法##
{% highlight java %}

	//AutoCompleteTextView的setAdapter方法，该Adapter需实现ListAdapter和Filterable接口
	//而ArrayAdapter是实现了这些接口的Adapter类
    public <T extends ListAdapter & Filterable> void setAdapter(T adapter) {
        if (mObserver == null) {
            mObserver = new PopupDataSetObserver(this);
        } else if (mAdapter != null) {
            mAdapter.unregisterDataSetObserver(mObserver);
        }
        mAdapter = adapter;
        if (mAdapter != null) {
            //noinspection unchecked
            mFilter = ((Filterable) mAdapter).getFilter();
            adapter.registerDataSetObserver(mObserver);
        } else {
            mFilter = null;
        }

        mPopup.setAdapter(mAdapter);
    }


{% endhighlight java %}


##二、ArrayAdapter是如何使用Filter接口的##


{% highlight java %}
	//将以prefix开头的字符串添加到FilterResults中，这个操作在异步线程里
	//最终，会将FilterResults结果输出到ui线程
	private class ArrayFilter extends Filter {
		
		//此方法中执行的代码，在异步线程里
        @Override
        protected FilterResults performFiltering(CharSequence prefix) {
            FilterResults results = new FilterResults();

            if (mOriginalValues == null) {
                synchronized (mLock) {
                    mOriginalValues = new ArrayList<T>(mObjects);
                }
            }

            if (prefix == null || prefix.length() == 0) {
                ArrayList<T> list;
                synchronized (mLock) {
                    list = new ArrayList<T>(mOriginalValues);
                }
                results.values = list;
                results.count = list.size();
            } else {
                String prefixString = prefix.toString().toLowerCase();

                ArrayList<T> values;
                synchronized (mLock) {
                    values = new ArrayList<T>(mOriginalValues);
                }

                final int count = values.size();
                final ArrayList<T> newValues = new ArrayList<T>();

                for (int i = 0; i < count; i++) {
                    final T value = values.get(i);
                    final String valueText = value.toString().toLowerCase();

                    // First match against the whole, non-splitted value
                    if (valueText.startsWith(prefixString)) {
                        newValues.add(value);
                    } else {
                        final String[] words = valueText.split(" ");
                        final int wordCount = words.length;

                        // Start at index 0, in case valueText starts with space(s)
                        for (int k = 0; k < wordCount; k++) {
                            if (words[k].startsWith(prefixString)) {
                                newValues.add(value);
                                break;
                            }
                        }
                    }
                }

                results.values = newValues;
                results.count = newValues.size();
            }

            return results;
        }

		//UI线程获取到FilterResults，并取出结果list，并调用notifyDataSetChanged()刷新数据
        @Override
        protected void publishResults(CharSequence constraint, FilterResults results) {
            //noinspection unchecked
            mObjects = (List<T>) results.values;
            if (results.count > 0) {
                notifyDataSetChanged();
            } else {
                notifyDataSetInvalidated();
            }
        }
    }

{% endhighlight java %}
