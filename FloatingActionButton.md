Material Design库源码分析--FloatingActionButton

Android support库最近终于放出了一系列material design support控件的官方实现，有了官方给出的实现，以后大家尽量还是使用官方的库吧。从这篇开始，我打算写一个系列，每一篇分析一个material design控件的实现方式，学习一下Google是如何在5.0之前的版本中实现material design的，以及Google是如何让一个控件同时兼容5.0和5.0之前的版本的。

第一篇分析的是FloatingActionButton

```java
public class FloatingActionButton extends ImageView {
```
FloatingActionButton集成自ImageView，因此你可以对它使用ImageView的各种属性，比如src

FloatingActionButton内部有一个FloatingActionButtonImpl类型的属性mImpl，FloatingActionButton其实是将几乎所有的方法都会调用这个mImpl对象的同名方法，说白了FloatingActionButton其实就是一个代理，真正用来控制整个控件显示的是这个mImpl对象。

查看构造函数，可以看到下面的代码：
```java
	if(VERSION.SDK_INT >= 21) {
        this.mImpl = new FloatingActionButtonLollipop(this, delegate);
    } else {
        this.mImpl = new FloatingActionButtonEclairMr1(this, delegate);
    }
```
可以看到针对api版本是否大于等于21，mImpl属性被new成了不同的对象。我们这里主要分析针对api 21之前的实现，也就是FloatingActionButtonEclairMr1。另外构造函数里传递的delegate对象主要是用来会掉FloatingActionButton的三个方法，这里给出创建delegate对象的代码：
```java
	ShadowViewDelegate delegate = new ShadowViewDelegate() {
	    public float getRadius() {
	        return (float)FloatingActionButton.this.getSizeDimension() / 2.0F;
	    }

	    public void setShadowPadding(int left, int top, int right, int bottom) {
	        FloatingActionButton.this.mShadowPadding.set(left, top, right, bottom);
	        FloatingActionButton.this.setPadding(left + FloatingActionButton.this.mContentPadding, top + FloatingActionButton.this.mContentPadding, right + FloatingActionButton.this.mContentPadding, bottom + FloatingActionButton.this.mContentPadding);
	    }

	    public void setBackgroundDrawable(Drawable background) {
	        FloatingActionButton.super.setBackgroundDrawable(background);
	    }
	};
```

在创建完mImpl对象后，会调用mImpl对象的几个方法:
```java
	this.mImpl.setBackgroundDrawable(background, this.mBackgroundTint, this.mBackgroundTintMode, this.mRippleColor, this.mBorderWidth);
    this.mImpl.setElevation(elevation);
    this.mImpl.setPressedTranslationZ(pressedTranslationZ);
```
查看FloatingActionButtonEclairMr1类的setBackgroundDrawable方法，在最后可以看到这几行代码：
```java
	this.mShadowDrawable = new ShadowDrawableWrapper(this.mView.getResources(), new LayerDrawable(layers), this.mShadowViewDelegate.getRadius(), this.mElevation, this.mElevation + this.mPressedTranslationZ);
    this.mShadowDrawable.setAddPaddingForCorners(false);
    this.mShadowViewDelegate.setBackgroundDrawable(this.mShadowDrawable);
```
其中mShadowViewDelegate就是我们前面讲的ShadowViewDelegate对象，this.mShadowViewDelegate.setBackgroundDrawable(this.mShadowDrawable)会调用FloatingActionButton.super.setBackgroundDrawable(background)方法，最终调用了ImageView的setBackgroundDrawable方法来显示一个drawable。

this.mShadowViewDelegate.setBackgroundDrawable(this.mShadowDrawable)中传递的mShadowDrawable其实是一个ShadowDrawableWrapper对象。而ShadowDrawableWrapper继承自DrawableWrapper，DrawableWrapper类继承自Drawable，它其实就是对Drawble的一个包装，它所有的方法都会调用内部的一个drawable对象的相同方法。ShadowDrawableWrapper多做了一点工作就是通过drawShadow方法给drawable添加一层阴影，drawShadow方法的实现其实很有意思，我留到以后再写吧，感兴趣的同学可以自己看看。

再看ShadowDrawableWrapper对象的创建this.mShadowDrawable = new ShadowDrawableWrapper(this.mView.getResources(), new LayerDrawable(layers), this.mShadowViewDelegate.getRadius(), this.mElevation, this.mElevation + this.mPressedTranslationZ);
我们可以看到这里是传递了一个LayerDrawable对象给ShadowDrawableWrapper，也就是这个LayerDrawable对象才是最终显示的。
```java
Drawable[] layers;
if(borderWidth > 0) {
    this.mBorderDrawable = this.createBorderDrawable(borderWidth, backgroundTint);
    layers = new Drawable[]{this.mBorderDrawable, this.mShapeDrawable, this.mRippleDrawable};
} else {
    this.mBorderDrawable = null;
    layers = new Drawable[]{this.mShapeDrawable, this.mRippleDrawable};
}
```
LayerDrawable无非就是包含了一个用来显示图像的Drawable，一个添加了边框的Drawable和一个显示水波效果的mRippleDrawable。我比较感兴趣的是rippleDrawable是怎么做到的，看代码：
```java
	GradientDrawable touchFeedbackShape = new GradientDrawable();
    touchFeedbackShape.setShape(1);
    touchFeedbackShape.setColor(-1);
    touchFeedbackShape.setCornerRadius(this.mShadowViewDelegate.getRadius());
    this.mRippleDrawable = DrawableCompat.wrap(touchFeedbackShape);
    DrawableCompat.setTintList(this.mRippleDrawable, createColorStateList(rippleColor));
    DrawableCompat.setTintMode(this.mRippleDrawable, Mode.MULTIPLY);
```
其实就是一个简单的GradientDrawable，然后使用DrawableCompat的setTintList，给这个GradientDrawable的不同选中状态设置不同的颜色。另外当FloatingActionButton被选中的时候，会通过一个Animation对象来改变阴影的大小，从而实现视觉上的z轴移动的效果。


感觉说的有点乱，大家将就看啦。
其实主要的就是一个ShadowDrawableWrapper类，另外就是DrawableCompat提供的一些辅助方法。