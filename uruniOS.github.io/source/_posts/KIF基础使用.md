---
title: KIF基础使用
date: 2017-02-07 16:15:05
tags:  
author: dsh
---

注：刚接触KIF,以下为我用到的方法,纯属个人浅层理解

### 方法命名

方法的命名必须以test开头，测试运行时，系统会寻找所有以“test”开头的方法，按照字母顺序运行，我一般以“test00”、“test01”……作为方法命名开头

### API介绍

- \- (void)beforeAll; 

beforeAll 是一个实际上只是在所有测试运行之前被调用一次的特殊方法. 你可以为你这里运行的测试设置任何实体变量和初始化条件.

- \- (void)afterAll;

afterAll 是在所有测试运行完成后被调用一次的特殊方法

#### 视图点击
最常用的方法为：

- \- (void)tapViewWithAccessibilityLabel:(NSString *)label;

它使用在给定的可访问标签上模拟在视图上的触击. 在大多数情况下，可访问标签都是匹配诸如按钮这种组件的可视的文本标签，否则则需要手动设置访问标签

eg.

设置访问标签

	[btn setAccessibilityLabel:@"test"];

设置视图点击

	[tester tapViewWithAccessibilityLabel:@"test"];

#### tableview相关测试

##### 测试row的点击
- \- (void)tapRowAtIndexPath:(NSIndexPath *)indexPath inTableViewWithAccessibilityIdentifier:(NSString *)identifier;

eg.

	- (void)testTapRow
	{
	    NSIndexPath *cellPath = [NSIndexPath indexPathForRow:0 inSection:0];
	    [tester tapRowAtIndexPath:cellPath inTableViewWithAccessibilityIdentifier:@"TableView"];
	}

 
- \- (void)tapRowAtIndexPath:(NSIndexPath *)indexPath inTableView:(UITableView *)tableView;

eg.

	- (void)testTapRow
	{
	    UITableView *tableView;
	    [tester waitForAccessibilityElement:NULL view:&tableView withIdentifier:@"TableView" tappable:NO];
	    [tester waitForAnimationsToFinish];
	    NSIndexPath *cellPath = [NSIndexPath indexPathForRow:0 inSection:0];
	    [tester tapRowAtIndexPath:cellPath inTableView:tableView];
	}
	
	以下方法为等待视图出现，默认超时未10s
	- (void)waitForAccessibilityElement:(UIAccessibilityElement **)element view:(out UIView **)view withIdentifier:(NSString *)identifier tappable:(BOOL)mustBeTappable;
	
##### 测试刷新
- \- (void)pullToRefreshViewWithAccessibilityLabel:(NSString *)label pullDownDuration:(KIFPullToRefreshTiming) pullDownDuration;

eg.

	[tester pullToRefreshViewWithAccessibilityLabel:@"TableView" pullDownDuration:KIFPullToRefreshInAboutOneSecond];

##### 测试滑动
horizontalFraction:水平方向 verticalFraction:垂直方向  
数值为-1~1，越接近0，速度越慢

- \- (void)scrollViewWithAccessibilityIdentifier:(NSString *)identifier byFractionOfSizeHorizontal:(CGFloat)horizontalFraction vertical:(CGFloat)verticalFraction;

eg.

	[tester scrollViewWithAccessibilityIdentifier:@"TableView" byFractionOfSizeHorizontal:0 vertical:-0.2];
	
##### cell左滑删除
- \- (void)swipeViewWithAccessibilityLabel:(NSString *)label inDirection:(KIFSwipeDirection)direction;

eg.

	[tester swipeViewWithAccessibilityLabel:@"Section 0 Row 0" inDirection:KIFSwipeDirectionLeft];
	[tester tapViewWithAccessibilityLabel:@"Delete"];

	
#### slider设值
- \- (void)setValue:(float)value forSliderWithAccessibilityLabel:(NSString *)label;

eg.

	  [tester setValue:2 forSliderWithAccessibilityLabel:@"Slider"];

#### switch开关
- \- (void)setOn:(BOOL)switchIsOn forSwitchWithAccessibilityLabel:(NSString *)label;

eg.

	[tester setOn:NO forSwitchWithAccessibilityLabel:@"Switch"];

#### picker
- \- (void)selectPickerViewRowWithTitle:(NSString *)title;

eg.

	[tester selectPickerViewRowWithTitle:@"Charlie"];



#### 文本输入
- \- (void)enterText:(NSString *)text intoViewWithAccessibilityLabel:(NSString *)label;

eg.

	[tester enterText:@"内容" intoViewWithAccessibilityLabel:@"TextField"];
	
#### 等待视图出现
- \- (UIView *)waitForViewWithAccessibilityLabel:(NSString *)label;

eg.

	[tester waitForViewWithAccessibilityLabel:@"TextField"];

#### 访问权限允许
- \- (BOOL)acknowledgeSystemAlert;

#### 延时
- \- (void)waitForTimeInterval:(NSTimeInterval)timeInterval;

eg.

	[tester waitForTimeInterval:3.0f];
	
#### 设置超时时间
- \- (instancetype)usingTimeout:(NSTimeInterval)executionBlockTimeout;

eg.

	[[tester usingTimeout:20] waitForViewWithAccessibilityLabel:@"TableView"];





