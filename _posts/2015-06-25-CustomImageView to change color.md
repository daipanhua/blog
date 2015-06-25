---

published: true
title: My first Jekyll theme Freshman
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

    private final int color;
    private final int colorFadeType;
    private float degrade;


    public CustomImageview(Context context) {
        this(context, null, 0);

    }

    public CustomImageview(Context context, AttributeSet attrs) {
        this(context, attrs, 0);

    }

    public CustomImageview(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);

        TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.customImageview);
		//获取指定的color，作为按压效果的颜色，通过做运算换算出颜色矩阵的值
        color = array.getColor(R.styleable.customImageview_customImageViewColor, 0);
		//颜色变化类型：
		// -1未指定，将保持原ImageView的效果
		// 1 颜色变深
		// 2 变成指定颜色
		// 4 颜色变浅
        colorFadeType = array.getInt(R.styleable.customImageview_colorFadeType, -1);
		// 颜色变化的程度
        degrade = array.getFloat(R.styleable.customImageview_degrade, 0.9f);

        array.recycle();
    }

	//因为项目中，是配合ImageLoader来加载图片的，而ImageLoader最终通过setImageBitmap来展现图片
	//因此在此处重写此方法。
    @Override
    public void setImageBitmap(Bitmap bmpOriginal) {
        if (colorFadeType == -1) {
            super.setImageBitmap(bmpOriginal);
            return;
        }
        if (color != 0 && colorFadeType != 2) {
            throw new IllegalArgumentException("ColorFadeType should set to type of 'color'");
        }
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
		//获取R G B的 255值
        int colorR = (color >> 16) & 0xff;
        int colorG = (color >> 8) & 0xff;
        int colorB = color & 0xff;
		//颜色变化矩阵
        float[] colorArray = new float[]{rgbRate, 0, 0, 0, colorR,
                0, rgbRate, 0, 0, colorG,
                0, 0, rgbRate, 0, colorB,
                0, 0, 0, aTransparency, 0};

        int width, height;
        height = bmpOriginal.getHeight();
        width = bmpOriginal.getWidth();

        Bitmap bmpGrayscale = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
        Canvas c = new Canvas(bmpGrayscale);
        Paint paint = new Paint();
        ColorMatrix cm = new ColorMatrix();
        cm.set(colorArray);
        ColorMatrixColorFilter f = new ColorMatrixColorFilter(cm);
        paint.setColorFilter(f);
        c.drawBitmap(bmpOriginal, 0, 0, paint);
        StateListDrawable sd = new StateListDrawable();
        //注意该处的顺序，只要有一个状态与之相配，背景就会被换掉
        //所以不要把大范围放在前面了，如果sd.addState(new[]{},normal)放在第一个的话，就没有什么效果了
        sd.addState(new int[]{android.R.attr.state_pressed}, new BitmapDrawable(getContext().getResources(), bmpGrayscale));
        sd.addState(new int[]{}, new BitmapDrawable(getContext().getResources(), bmpOriginal));
        setImageDrawable(sd);
    }

}
{% endhighlight java %}