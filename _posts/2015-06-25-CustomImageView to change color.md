---

published: true
title: 给ImageView自动加上按压效果(前景色，非背景) 
layout: post
author: 代盼华
category: 技术
tags: 
- Android
- ImageView
- Color
description: 给ImageView自动加上按压效果 

---

#功能#
在App开发中，我们经常会用到有图标的按钮，而有时候设计并没有给图标指定按压或选中的效果，或者图标是从网络获取而来的。为了能在xml中直接指定图标的按压效果，通过自定义的ImageView来实现此功能。


#技术实现#
主要用到了是`android.graphics.ColorMatrix`这个类，通过颜色矩阵对原图片的颜色进行修改，而生成新的BitMap,并将其添加到StateListDrawable中。

#实现细节#

{% highlight java %}
public class CustomImageview extends ImageView {

	private int color = 0;
	private int colorFadeType = -1;
	private float degrade;

	public CustomImageview(Context context) {
		this(context, null, 0);

	}

	public CustomImageview(Context context, AttributeSet attrs) {
		this(context, attrs, 0);

	}

	public CustomImageview(Context context, AttributeSet attrs, int defStyleAttr) {
		super(context, attrs, defStyleAttr);

		TypedArray array = context.obtainStyledAttributes(attrs,
				R.styleable.customImageview);
		// color做运算换算出颜色矩阵的值
		color = array.getColor(
				R.styleable.customImageview_customImageViewColor, 0);
		colorFadeType = array.getInt(R.styleable.customImageview_colorFadeType,
				-1);
		degrade = array.getFloat(R.styleable.customImageview_degrade, 0.9f);
		array.recycle();
		setImageDrawable(getDrawable());
	}

	@Override
	public void setImageBitmap(Bitmap bmpOriginal) {
		if (colorFadeType != 1 && colorFadeType != 2 && colorFadeType != 4) {
			setDrawable(new BitmapDrawable(getResources(), bmpOriginal));
			return;
		}
		if (color != 0 && colorFadeType != 2) {
			throw new IllegalArgumentException(
					"ColorFadeType should set to type of 'color'");
		}
		// 变深用比率，变浅用透明度
		if (degrade > 0.9) {
			degrade = 0.9f;
		}
		float rgbRate = 1.0f;
		float aTransparency = 1.0f;
		if (colorFadeType == 1) {
			aTransparency = 0.5f;
		} else if (colorFadeType == 2) {
			rgbRate = 0;
		} else if (colorFadeType == 4) {
			rgbRate = degrade;
		}

		int colorR = (color >> 16) & 0xff;
		int colorG = (color >> 8) & 0xff;
		int colorB = color & 0xff;

		float[] colorArray = new float[] { rgbRate, 0, 0, 0, colorR, 0,
				rgbRate, 0, 0, colorG, 0, 0, rgbRate, 0, colorB, 0, 0, 0,
				aTransparency, 0 };

		int width, height;
		height = bmpOriginal.getHeight();
		width = bmpOriginal.getWidth();

		Bitmap bmpGrayscale = Bitmap.createBitmap(width, height,
				Bitmap.Config.ARGB_8888);
		Canvas c = new Canvas(bmpGrayscale);
		Paint paint = new Paint();
		ColorMatrix cm = new ColorMatrix();
		cm.set(colorArray);
		ColorMatrixColorFilter f = new ColorMatrixColorFilter(cm);
		paint.setColorFilter(f);
		c.drawBitmap(bmpOriginal, 0, 0, paint);

		StateListDrawable sd = new StateListDrawable();
		// 注意该处的顺序，只要有一个状态与之相配，背景就会被换掉
		// 所以不要把大范围放在前面了，如果sd.addState(new[]{},normal)放在第一个的话，就没有什么效果了
		sd.addState(new int[] { android.R.attr.state_pressed },
				new BitmapDrawable(getContext().getResources(), bmpGrayscale));
		sd.addState(new int[] {}, new BitmapDrawable(getContext()
				.getResources(), bmpOriginal));
		setDrawable(sd);
	}

	@Override
	public void setImageDrawable(Drawable drawable) {
		this.setImageBitmap(((BitmapDrawable) drawable).getBitmap());
	}

	private void setDrawable(Drawable d) {
		Class clazz = ImageView.class;
		try {
			Method m = clazz
					.getDeclaredMethod("updateDrawable", Drawable.class);
			m.setAccessible(true);
			m.invoke(this, d);
		} catch (NoSuchMethodException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (IllegalArgumentException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (IllegalAccessException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (InvocationTargetException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		requestLayout();
		invalidate();
	}

	@Override
	public void setImageResource(int resId) {
		super.setImageResource(resId);
		setImageDrawable(getDrawable());
	}
}
}
{% endhighlight java %}

不足之处，希望大家能在评论中指正 ~~

demo下载连接：[百度网盘](http://pan.baidu.com/s/1jGpcI3c){:target="_blank"}