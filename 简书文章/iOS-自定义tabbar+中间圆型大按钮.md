最近公司想做一个中间突出的按钮，做的过程中发现突出的地方总是点不中，所以记录一下。
代码：在tabbar上面添加5个按钮,中间的frame设置大一点，button添加点击事件和平时相同。然后重写点击函数在自己新建的tabbar中重写`-(UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event`，
代码：
```
-(UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    UIView * view = nil;
    if (self.isHidden == NO) {
        for (UIView *subView in self.subviews) {
            CGPoint testPoint = [subView convertPoint:point fromView:self];//转换坐标
//判断是否view是否包含点击事件
            if (CGRectContainsPoint(subView.bounds, testPoint)) {
//self.centerButton是中间的按钮,判断点击point是否在中间的按钮范围内
                if (CGRectContainsPoint(self.centerButton.frame,point)) {
                    return self.centerButton;
                } else if ([subView isKindOfClass:[UIButton class]]) {
//判断 其他的四个按钮是否接受点击事件
                    view = subView;
                    return view;
                }else {
//除了按钮之外，就让父视图去处理事件
                    UIView * superView=[super hitTest:point withEvent:event];
                    return superView;
                }
            }
        }
//返回nil是不处理
        return nil;
        
    }else{
        return nil;
    }
}
```
### 添加按钮到tabbar
```
- (void)setMMDItems
{
    for (UIButton* button in _buttons) {
        [button removeActionCompletionBlocksForControlEvents:UIControlEventTouchUpInside];
        [button removeFromSuperview];
    }
    for (UILabel *label in _labels) {
        [label removeFromSuperview];
    }
    _buttons = [NSMutableArray array];
    _labels = [NSMutableArray array];
    _badges = [NSMutableArray array];

    int btnNum = 5;
    for (int i = 0; i < btnNum; i++) {
        UIButton* button = [UIButton buttonWithType:UIButtonTypeCustom];
        UIImage* buttonImage = [self getImageWithTabIndex:i isPressed:NO];
        UIImage* buttonPressedImage = [self getImageWithTabIndex:i isPressed:YES];
        button.imageEdgeInsets = UIEdgeInsetsMake(-11, 0, 0, 0);
        [button setImage:buttonImage forState:UIControlStateNormal];
        [button setImage:buttonImage forState:UIControlStateDisabled];
        [button setImage:buttonPressedImage forState:UIControlStateHighlighted];
        [button setImage:buttonPressedImage forState:UIControlStateSelected];
        [self addSubview:button];
       
        UILabel* titleLabel = [[UILabel alloc] initWithFrame:CGRectZero];
        titleLabel.contentMode = UIViewContentModeTop;
        titleLabel.backgroundColor = [UIColor clearColor];
        titleLabel.textColor = COLORRGB(149, 149, 149);
        titleLabel.textAlignment = NSTextAlignmentCenter;
        titleLabel.font = [UIFont systemFontOfSize:10];
        titleLabel.text = [self getTabbarTitleWithTabIndex:i];
        [self addSubview:titleLabel];
        
        //CGFloat textOffset = 14;
        int firstBtnWidth = kScreenWidth/btnNum;
       
        UILabel *badge = [[UILabel alloc]init];
        badge.layer.cornerRadius = 3.5;
        badge.backgroundColor = [UIColor redColor];
        badge.layer.masksToBounds = YES;
        badge.hidden = YES;
        [self addSubview:badge];
        
        badge.frame = CGRectMake((i+1)*firstBtnWidth - 15 - 20, 7, 7, 7);
        CGFloat height = 0.0f;//按钮高度49
        height = 49.0f;
        if (i == 2) {
            button.frame = CGRectMake(i*firstBtnWidth, -20, firstBtnWidth, height);
            self.centerButton = button;
            self.centerButton.acceptEventInterval = 1.5;//点击频率间隔1.5s
        }else {
            button.frame = CGRectMake(i*firstBtnWidth, 0, firstBtnWidth, height);
        }
        titleLabel.frame = CGRectMake(i*firstBtnWidth, 35, firstBtnWidth, 12);
        
        [button addActionCompletionBlock:^(id sender) {
            self.selectedIndex = i;
        } forControlEvents:UIControlEventTouchUpInside];
        
        if (i == self.selectedIndex) {
            [button setImage:[button imageForState:UIControlStateSelected] forState:UIControlStateNormal];
             titleLabel.textColor = COLORRGB(59, 171, 179);
        }  
        [_buttons addObject:button];
        [_labels addObject:titleLabel];
        [_badges addObject:badge];
    }
}
```
