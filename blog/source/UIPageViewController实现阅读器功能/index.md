title: UIPageViewController实现阅读器功能
date: 2023-07-25 13:59:03
---
首先，UIPageViewController坑很多，下面文章都有描述

[UIPageViewController缺陷](https://www.jianshu.com/p/3cca93ceee96)

#### 1、翻页方式

一共四种翻页方式（UIPageViewControllerTransitionStyle）

- 滑动翻页（UIPageViewControllerTransitionStyleScroll）
  - 点击翻页
  - 手势翻页
- 仿真翻页（UIPageViewControllerTransitionStylePageCurl）
  - 点击翻页
  - 手势翻页

手势翻页直接设置好UIPageViewControllerTransitionStyle后滑动就可以实现。点击翻页通过UIPageViewController子页面添加点击手势UIPanGestureRecognizer实现

#### 2、手势翻页

手势主要通过的UIPageViewController代理实现

```objective-c
- (nullable UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerBeforeViewController:(UIViewController *)viewController;

- (nullable UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerAfterViewController:(UIViewController *)viewController;

```

注意：这两种方法属于提供页面数据源的代理，并不是页面翻到前一页、后一页的回调

所以无法通过这两个方法来设置页面的页码数

UIPageViewControllerTransitionStylePageCurl模式下可以通过从pageViewController取得当前页面再根据是after或是before将页码加减。

但是UIPageViewControllerTransitionStyleScroll模式下，你手势翻页上面两种方法会调用多次，它其实是一种数据源的代理，例如你从第二页翻到第三页，它会要求你提供第四页的数据。因为手势翻页需要提前准备好下一页的数据才能实现平滑互动的效果。从我实验结果看，这种模式手动翻页，以上方法会调用三次，会调用两次after方法，一次before方法。所以你无法通过这两种方法来获取当前滑动到了第几页。

UIPageViewControllerTransitionStyleScroll模式下使用下面两个方法实现获取当前页面

```objective-c
- (void)pageViewController:(UIPageViewController *)pageViewController didFinishAnimating:(BOOL)finished previousViewControllers:(NSArray *)previousViewControllers transitionCompleted:(BOOL)completed;

```

通过pageViewController获取当前页面的页码page

#### 3、点击翻页

点击翻页需要调取setViewControllers方法

```objective-c
    dispatch_async(dispatch_get_main_queue(), ^{
        [self.pageViewController setViewControllers:viewControllers direction:direction animated:animated completion:^(BOOL finished) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) {
                    completion(finished);
                }
            });
        }];
    });

```

注意：一定要在主线程中执行，否则会发生crash(报错Unbalanced calls to begin/end appearance transitions for )

setViewControllers参数是一个数组，如果当时是滑动翻页模式时（UIPageViewControllerTransitionStyleScroll）viewControllers只需一个@[nextVc]

而处在仿真翻页模式时（UIPageViewControllerTransitionStylePageCurl）viewControllers需要两个@[currentVc,nextVc]