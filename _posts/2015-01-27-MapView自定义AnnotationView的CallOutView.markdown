---
layout: post
title:  "MapView自定义AnnotationView的CallOutView"
date:   2015-01-27 09:32:24
categories: iOS开发
---
在一些关于地图的开发的时候通常情况下都会自定义点击地图位置后的弹出框效果，如果你使用原生的AnnotationView的话你会发现你能自定义的只有这个

{% highlight Objective-c %}
// The left accessory view to be used in the standard callout.
@property (strong, nonatomic) UIView *leftCalloutAccessoryView;
// The right accessory view to be used in the standard callout.
@property (strong, nonatomic) UIView *rightCalloutAccessoryView;
{% endhighlight %}


也就是说差不多自定义出来的像这个样子

![image](/images/000002.png)

如果你想做出这样的效果

![image](/images/000001.png)

就要使用下文的方法了  
  
  

**第一种方法: 把弹出框作为一个annotation view**  

这种方法的原理就是把callOutView作为AnnotationView来显示，是比较常见的做法，你可以参考[这篇博文](http://blog.csdn.net/mad1989/article/details/8794762)。



**第二种方法: 自定义annotation view**

当你点击annotation弹出call out view。其中的原理就是annotation view在选中状态时弹出了call out view。这里我们只需要自定义一个annotation view使其选中状态时添加一个subview就可以  
 
*   自定义annotation view
{% highlight Objective-c %}
@interface MLLMapAnnotationView : MKAnnotationView
@end  
{% endhighlight %}
{% highlight Objective-c %}
@implementation MLLMapAnnotationView {
    BOOL customCallOutDisplayed;
    MLLMapAnnotationCallOutView *_callOutView;
}
- (instancetype)initWithAnnotation:(id<MKAnnotation>)annotation reuseIdentifier:(NSString *)reuseIdentifier
{
    self = [super initWithAnnotation:annotation reuseIdentifier:reuseIdentifier];
    if (self) {
        self.canShowCallout = NO;
        self.image = [UIImage imageNamed:@"map_icon_address_red"];
    }
    return self;
}
- (void)setSelected:(BOOL)selected animated:(BOOL)animated
{
    [super setSelected:selected animated:animated];
    if (selected) {
        self.image = [UIImage imageNamed:@"map_icon_address_blue"];
    }else {
        self.image = [UIImage imageNamed:@"map_icon_address_red"];
    }
    if (!_callOutView) {
        [self setupCallOut];
    }
    if (_callOutView.superview != self && selected) {
        [self addSubview:_callOutView];
        _callOutView.alpha = 0;
        [UIView animateWithDuration:animated ? 0.3 : 0 animations:^{
            _callOutView.alpha = 1;
        }];
        
        if ([self.annotation isKindOfClass:[MLLMapAnnotation class]]) {
            MLLMapAnnotation *anno = self.annotation;
            _callOutView.nameLabel.text = anno.name;
            _callOutView.addressLabel.text = [NSString stringWithFormat:@"地址：%@", anno.address];
            _callOutView.phoneLabel.text = [NSString stringWithFormat:@"电话：%@", anno.phone];
            _callOutView.distanceLabel.text= anno.subtitle;
        }
    }
    if (!selected && _callOutView.superview) {
        [UIView animateWithDuration:animated ? 0.3 : 0 animations:^{
            _callOutView.alpha = 0;
        } completion:^(BOOL finished) {
            [_callOutView removeFromSuperview];
        }];
    }
}
- (void)setupCallOut
{
    UINib *nib = [UINib nibWithNibName:@"MLLMapAnnotationCallOutView" bundle:nil];
    NSArray *views = [nib instantiateWithOwner:nil options:nil];
    _callOutView = views.firstObject;
    _callOutView.frame = CallOutFrame;
    _callOutView.delegate = self;
}
{% endhighlight %}
*在这里在setSelected方法内部直接加载了call out view,这里我是为了偷懒写成这样。这里的正确写法是应该用一个view controller来控制call out view,所以这里的selected方法才看起>来比较繁复。*  
*MLLMapAnnotationCallOutView是我已经写好了的call out view的类。*  

*   触摸事件处理：因为这里的call out view被addSubView到了annotation view的外部，这样的话call out view是收不到触摸事件的，所以我们还要让他收的到触摸事件：重写annotation view的hitTest:withEvent:。  

{% highlight Objective-c %}
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event 
{
    UIView *hitTest = [super hitTest:point withEvent:event];
    if (!hitTest && self.selected) {
        CGPoint insidepoint = [_callOutView convertPoint:point fromView:self];
        hitTest = [_callOutView hitTest:insidepoint withEvent:event];
    }
    return hitTest;
}
{% endhighlight %}

好了，上图的自定义目标就完成了。
