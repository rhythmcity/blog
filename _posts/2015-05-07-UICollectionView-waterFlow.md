---
layout: post
title: "用UICollectionView实现瀑布流效果"
date: 2015-05-07 11:57:00
categories: iOS
featured_image: /images/cover.jpg
---

最终效果是这样

![最终效果](https://github.com/rhythmcity/rhythmcity.github.io/raw/master/images/waterFlow/Effect.png)


UIcollectionView 是iOS6新加入的UIkit组件，
用法和UITableview差不多同样是要实现两个代理协议

```objective-c
    @property (nonatomic, assign) id <UICollectionViewDelegate> delegate;
    @property (nonatomic, assign) id <UICollectionViewDataSource> dataSource;
```

首先创建一个CollectView

```objective-c
self.conllectionFlowLayout = [[UICollectionViewFlowLayout alloc] init];
self.conllectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds collectionViewLayout:self.conllectionFlowLayout];
self.conllectionView.backgroundColor = [UIColor whiteColor];
[self.conllectionView registerClass:[CollectionViewCell class] forCellWithReuseIdentifier:@"cellID"];

self.conllectionView.delegate = self;
self.conllectionView.dataSource = self;
[self.view addSubview:self.conllectionView];
```
然后实现几个代理，这根tableview 的写法基本一样

```objective-c

#pragma mark UICollectionViewDataSource
- (NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section{

   return 30;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{

    static NSString *cellID = @"cellID";
    CollectionViewCell *cell = [self.conllectionView dequeueReusableCellWithReuseIdentifier:cellID forIndexPath:indexPath];
    cell.backgroundColor = [UIColor grayColor];
    return cell;
}

#pragma mark UICollectionViewDelegate
- (CGFloat)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout*)collectionViewLayout minimumInteritemSpacingForSectionAtIndex:(NSInteger)section{

   return 0;
}

- (CGSize)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout*)collectionViewLayout sizeForItemAtIndexPath:(NSIndexPath *)indexPath{

   CGFloat randomHeight = 100+(arc4random()%140);//随机返回高度
   return CGSizeMake((self.conllectionView.bounds.size.width/3)-10, randomHeight);
}

```
实现了这些现在的样子是这样的
![](https://github.com/rhythmcity/rhythmcity.github.io/raw/master/images/waterFlow/nomal.png)

下面我们要怎么把他做成我们以前想要的样子呢 那就是UICollectionView的灵魂，` "UICollectionViewLayout" `
这时候我们创建一个MasonyLayout 继承自UICollectionViewLayout ，然后定义一个协议用来接受每一个item的高度，并且声明两个成员变量一个是 "列数" 另一个是“每列的间隔”

```objective-c

#import <UIKit/UIKit.h>
@class MasonyLayout;

@protocol MasonryViewLayoutDelegate <NSObject>
@required
- (CGFloat) collectionView:(UICollectionView*) collectionView
layout:(MasonyLayout*) layout
heightForItemAtIndexPath:(NSIndexPath*) indexPath;
@end

@interface MasonyLayout : UICollectionViewLayout
@property(nonatomic,assign)NSInteger numberofColumns;
@property(nonatomic,assign)CGFloat interItemSpacing;
@property (weak, nonatomic) id<MasonryViewLayoutDelegate> delegate;
@end

```
然后在.m文件中实现 `-(void)prepareLayout` 方法 每次重新给出layout时都会调用prepareLayout，这样在以后如果有collectionView大小变化的需求时也可以自动适应变化。

```objective-c
-(void)prepareLayout{
self.lastYValueForColumn = [NSMutableDictionary dictionary];//存储每列最后一个的item信息 用来计算下一列起始位置
CGFloat currentColumn = 0;
CGFloat fullwidth = self.collectionView.frame.size.width;
CGFloat availableSpaceExcludingPadding = fullwidth - self.interItemSpacing*(self.numberofColumns+1);
CGFloat itemWidth = availableSpaceExcludingPadding/self.numberofColumns;

self.layoutInfo = [NSMutableDictionary dictionary]; //用来存储每一个item 的布局信息
NSIndexPath *indexPath;
NSInteger numSections = [self.collectionView numberOfSections];

for(NSInteger section = 0; section < numSections; section++)  {

NSInteger numItems = [self.collectionView numberOfItemsInSection:section];
for(NSInteger item = 0; item < numItems; item++){
indexPath = [NSIndexPath indexPathForItem:item inSection:section];

UICollectionViewLayoutAttributes *itemAttributes =
[UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:indexPath];

CGFloat x = self.interItemSpacing + (self.interItemSpacing + itemWidth) * currentColumn;
CGFloat y = [self.lastYValueForColumn[@(currentColumn)] doubleValue];
/**
通过代理获取每一个item 的高度
*/
CGFloat height = [((id<MasonryViewLayoutDelegate>)self.collectionView.delegate)
collectionView:self.collectionView
layout:self
heightForItemAtIndexPath:indexPath];

itemAttributes.frame = CGRectMake(x, y, itemWidth, height);
y+= height;
y += self.interItemSpacing;

self.lastYValueForColumn[@(currentColumn)] = @(y);

currentColumn ++;
if(currentColumn == self.numberofColumns) currentColumn = 0;
self.layoutInfo[indexPath] = itemAttributes;
}
}
}

```

接下来实现 -(NSArray *)layoutAttributesForElementsInRect:(CGRect)rect  这个方法会返回rect中的所有的元素的布局属性
返回的是包含UICollectionViewLayoutAttributes的NSArray,我们现在所有的item的属性都放在 self.layoutInfo 这个字典里面所以我们从这里面取出我们的layoutAttributes

```objective-c

- (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect {

NSMutableArray *allAttributes = [NSMutableArray arrayWithCapacity:self.layoutInfo.count];
[self.layoutInfo enumerateKeysAndObjectsUsingBlock:^(NSIndexPath *indexPath,
UICollectionViewLayoutAttributes *attributes, BOOL *stop) {
   if (CGRectIntersectsRect(rect, attributes.frame)) {
      [allAttributes addObject:attributes];
  }
}];
return allAttributes;
}

```

到目前为止我们的布局和每一个item的样式都搞定了，运行起来看一下,你会发现什么都没有，那是因为我们还差一个方法

```
- (CGSize)collectionViewContentSize; // Subclasses must override this method and use it to return the width and height of the collection view’s content. These values represent the width and height of all the content, not just the content that is currently visible. The collection view uses this information to configure its own content size to facilitate scrolling.
```
看苹果官方的文档说明，子类必须重写这个方法返回content view的宽和高 这个高度是所有的宽和高不仅仅是当前屏幕的，collectionView需要这个信息来滚动 其实就是给collectionView 一个contentSize  因为父类也是UIScrollview

```
//计算collectionview 的contentSize
-(CGSize) collectionViewContentSize {

NSUInteger currentColumn = 0;
CGFloat maxHeight = 0;
do {
CGFloat height = [self.lastYValueForColumn[@(currentColumn)] doubleValue];
if(height > maxHeight)
maxHeight = height;
currentColumn ++;
} while (currentColumn < self.numberofColumns);

return CGSizeMake(self.collectionView.frame.size.width, maxHeight);
}

```
这样我们的MasonyLayout 就写好了，这样我们在初始化UICollectionView时 就可以用我们自定义的布局了 并且实现我们自定义的协议：

```objective-c

self.masonyLayout = [[MasonyLayout alloc] init];
self.masonyLayout.numberofColumns = 3;
self.masonyLayout.interItemSpacing = 12.5;
self.masonyLayout.delegate = self;
self.conllectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds collectionViewLayout:self.masonyLayout];



#pragma mark MasonryViewLayoutDelegate
- (CGFloat) collectionView:(UICollectionView*) collectionView
layout:(MasonyLayout*) layout
heightForItemAtIndexPath:(NSIndexPath*) indexPath{


CGFloat randomHeight = 100+(arc4random()%140);

return randomHeight;
}

```

这样就实现了我们最开始想好的效果，只是因为瀑布流的样式，在应用开发里面很常见，常用于图片浏览，其实UICollectionView可以实现很多酷炫的布局方式，还需要多研究研究









