---
layout:     post
title:      "实现一款酷炫的iOS进度条(二)"
date:       2017-1-23 8:00:00
author:     "Karthus"
header-img: "img/post_header_bg.jpg"
tags:
    - CoreAnimation
    - iOS动画
    - 进度条
---

- [写一个酷炫的iOS进度条动画](http://www.jianshu.com/p/4e26241e5211)
- CoreAnimation笔记(之后写一下)


前面这篇文章里讲了大致的实现思路，这篇文章一步步实现一个进度条。就以上篇文章提到的第二个进度条为例。

原效果：
![进度条11.gif](http://upload-images.jianshu.io/upload_images/1249505-72925479cd14b2a8.gif?imageMogr2/auto-orient/strip)

这个动画效果明显比铅笔那个要简单，实现起来效果也更好，动画更加平滑，还是按照老思路截几个关键帧的图来慢放一下动画。


![1.png](http://upload-images.jianshu.io/upload_images/1249505-1c791bebe62127e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![2.png](http://upload-images.jianshu.io/upload_images/1249505-cfbbb2ab6ef9b577.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![3.png](http://upload-images.jianshu.io/upload_images/1249505-b832db6b3da1fcec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- **初始状态**，最开始我想的还是用两个shapeLayer来实现，但是经过观察后你会发现**白色箭头变化的过程是由上方的矩形和下方的三角形一起伸缩形变的**，若要平滑的实现这个效果就需要将箭头分解为两个shapeLayer，于是不如单独用一个view来放这两个layer，命名为**ZYProgressIndicator**

      @property (nonatomic, strong) ZYProgressIndicator *indicator;
      @property (nonatomic, strong) CAShapeLayer *containerLayer; // 深色方框
      @property (nonatomic, strong) CAShapeLayer *progressLayer; // 之后用来填充进度条的白色layer

- ** ZYProgressIndicator**由一个矩形和一个三角形组成，
依然用UIBezierPath画出这两个layer的形状

      @property (nonatomic, strong) CAShapeLayer *triangleLayer;
      @property (nonatomic, strong) CAShapeLayer *rectangleLayer;

- 由动画分解可以看出箭头的变化：**上方矩形横向拉伸，长边变为宽边，下方三角形则一直变小即可。**给两个layer分别添加形变动画：

      - (void)readyForDownload
      {
          self.progressLabel.hidden = NO;
          
          // 三角形x,y都缩小为原来的0.3倍
          CABasicAnimation *triAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
          triAnimation.toValue = [NSValue valueWithCATransform3D:CATransform3DMakeScale(0.3, 0.3, 1)];
          triAnimation.duration = duration;
          triAnimation.fillMode = kCAFillModeForwards;
          triAnimation.removedOnCompletion = NO;
          [self.triangleLayer addAnimation:triAnimation forKey:kTriangleReadyAnimationKey];
    
          //x轴拉长为1.5倍即宽边拉长为原来的1.5倍，y轴缩短为0.9
          CABasicAnimation *recAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
          recAnimation.toValue = [NSValue valueWithCATransform3D:CATransform3DMakeScale(1.5, 0.9, 1)];
          recAnimation.duration = duration;
          recAnimation.fillMode = kCAFillModeForwards;
          recAnimation.removedOnCompletion = NO;
          [self.rectangleLayer addAnimation:recAnimation forKey:kRectangleReadyAnimationKey];
    
      }

这样就完成了箭头变化的形变动画，就是这么简单。顺便把恢复原样的动画也写一下：
**只需要将toValue改为CATransform3DIdentity即可**,不再更详细赘述
        
    CABasicAnimation *resumeAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
    resumeAnimation.toValue = [NSValue valueWithCATransform3D:CATransform3DIdentity];

- 到此indicator中相关动画基本就完成了，接下来是主类中的**containerLayer**了，containerLayer的形变动画也是和indicator中一样，利用**CABasicAnimation**变化**transform**即可

- 将动画慢放就会发现indicator有个**先向上->落下->右移一小段->左移至起点的动画**，即是一个位移动画，利用**CAKeyframeAnimation**来变化**position**实现：

      /**
       indicator移动路径：先向上->落下->右移一小段->左移至起点
 
       @return indicator移动路径
       */
      - (UIBezierPath *)indicatorMoveToReadyPath
      {
          UIBezierPath *path = [UIBezierPath bezierPath];
          CGPoint startPoint = self.indicator.layer.position;
          // 先向上偏移40
          CGPoint topPoint = (CGPoint){startPoint.x,startPoint.y-40};
          // 落回原先位置，稍微向上微调让indicator的下角对准progress
          CGPoint horizentalPoint = (CGPoint){startPoint.x,startPoint.y-7.5};
          //向右移动一小段距离
          CGPoint rightPoint = (CGPoint){startPoint.x+20,horizentalPoint.y};
          //左移至起点
          CGPoint leftPoint = (CGPoint){0,horizentalPoint.y};
          [path moveToPoint:startPoint];
          [path addLineToPoint:topPoint];
          [path addLineToPoint:horizentalPoint];
          [path addLineToPoint:rightPoint];
          [path addLineToPoint:leftPoint];
          return path;
      }

- 位移过程中indicator是有角度的偏移的，此时利用CAAnimationGroup组合一下动画：

      - (CAAnimationGroup *)indicatorReadyAnimation
      {
          CAKeyframeAnimation *moveAnim = [CAKeyframeAnimation animationWithKeyPath:@"position"];
          moveAnim.path = [self indicatorMoveToReadyPath].CGPath;
    
          CAKeyframeAnimation *rotationAnim = [CAKeyframeAnimation animationWithKeyPath:@"transform.rotation.z"];
          rotationAnim.values = @[@(0),@(kHighSpeedAngle)];
          rotationAnim.keyTimes = @[@(0.4),@(0.7)];
          // 设置keytime让indicator大概在落下后再开始偏移，更符合原图动画效果

          CAAnimationGroup *group =[CAAnimationGroup animation];
          group.duration = 0.9;
          group.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
          group.animations = @[rotationAnim,moveAnim];
          group.fillMode = kCAFillModeForwards;
          group.removedOnCompletion = NO;
          return group;
      }

- 至此前半段动画基本完成，一些细节可以自己完善
- 进度条填充和indicator位移动画和上一篇文章提到的一样，不过此次我采用**CADisplayLink**来刷新进度条，**CADisplayLink是一个与屏幕刷新率保持同步的timer，可以让进度条动画更流畅，看起来更加舒服**，在适当的地方将displayLink添加进runloop即可。

      // 让displayLink一直调用此方法刷新进度条
      - (void)refreshUI
      {
          self.indicator.transform = CGAffineTransformMakeTranslation(CGRectGetWidth(self.frame)*_progress, 0);
          self.progressLayer.path = [self progressPathWithProgress:_progress].CGPath;    
      }

- **下载失败动画：**我的思路是将此处的失败动画分为两部分。
进度条类似水面下降动画和水流流下动画两部分。

**1.水面下降动画：利用bounds的变化来实现,将进度条高度直接变为0**

        - (CAAnimationGroup *)progressDissmissAnimation
        {
            // 将进度条高度变为0
            CABasicAnimation *scaleAnim = [CABasicAnimation animationWithKeyPath:@"transform"];
            scaleAnim.toValue = [NSValue valueWithCATransform3D:CATransform3DMakeScale(1, 0, 1)];
    
            // 由于layer的形变是以原先position为中间点变化
            // 所以要同步加上朝下位移动画，让“水波”看起来是紧贴着底部
            CABasicAnimation *downAnim = [CABasicAnimation animationWithKeyPath:@"position.y"];
            downAnim.toValue = @(self.progressLayer.position.y+kProgressWidth/2);
    
            CAAnimationGroup *group = [CAAnimationGroup animation];
            group.fillMode = kCAFillModeForwards;
            group.removedOnCompletion = NO;
            group.animations = @[scaleAnim,downAnim];
            return group;
        }

**2.水流流下动画：此处直接新添加一个layer用作水流，让layer做形变，长度变化，实现类似水流不断流出变长效果，再同时做向下位移动画形成水流不断流出并流下的效果**，思路大致是这样

        // 添加一个layer，在当前进度条的进度位置
        CALayer *downLayer = [CALayer layer];
        downLayer.frame = (CGRect){CGRectGetWidth(self.frame)*_progress,CGRectGetHeight(self.frame)/2,2,0};
        downLayer.backgroundColor = [UIColor whiteColor].CGColor;
        downLayer.cornerRadius = 2.5f;
        [self.layer addSublayer:downLayer];

 为此layer添加下述动画：

       /**
        progress持续延长  由于延长是由中心点到上下两边延长
        因此添加向下位移动画 使其看起来向单向下延伸
        @return 动画组
        */
         - (CAAnimationGroup *)progressDownAnimation
       {
           CABasicAnimation *scaleAnim = [CABasicAnimation animationWithKeyPath:@"bounds.size"];
           scaleAnim.toValue = [NSValue valueWithCGSize:(CGSize){1,CGRectGetWidth(self.frame)*_progress}];
    
           //进度条水面下降时 长度随之一起改变 并向下移
           CABasicAnimation *downAnim = [CABasicAnimation animationWithKeyPath:@"position.y"];
           downAnim.toValue = @((CGRectGetWidth(self.frame)*_progress)/2+3);
           downAnim.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
    
           CAAnimationGroup *group = [CAAnimationGroup animation];
           group.fillMode = kCAFillModeForwards;
           group.removedOnCompletion = NO;
           group.animations = @[scaleAnim,downAnim];
           return group;
       }
- 下载失败动画基本效果就完成了，只需再添加resume动画，整个进度条动画的主要动画就基本完成了。resume动画比较简单不再赘述有兴趣的朋友可以去看一下代码。


##**实现效果**

![](http://upload-images.jianshu.io/upload_images/1249505-55fd1c691c5ec689.gif?imageMogr2/auto-orient/strip)

![](http://upload-images.jianshu.io/upload_images/1249505-8937f747d9a5b91c.gif?imageMogr2/auto-orient/strip)

**下载过程中加速减速而变化indicator角度暂时想不到实现方法，有思路的朋友欢迎交流**

代码放在[github](https://github.com/Karthus1110/ZYDownloadProgress),如欢迎star
