---
layout: post
title:  "IOS auto layout 纯代码对糗事百科这种cell布局注意事项"
date:   2015-06-24 12:00:00
categories:  🍎ios

---

* content
{:toc}

### 1.建立一个cell继承至UITableViewCell
JJTableViewCellPic.h 

    typedef NS_ENUM(NSInteger, CellType) {
    CellTypeText = 0,
    CellTypeImg   = 1,
    CellTypeVideo  = 2,
    };
    @class Articles;
    @interface JJTableViewCellPic : UITableViewCell
    @property (assign, nonatomic) NSNumber *artcicle_id;
    @property (assign, nonatomic) NSNumber *user_id;
    @property (assign, nonatomic) CGSize   image_size;
    @property (assign, nonatomic) CellType   cell_type;
    @property (strong,nonatomic ) Articles *entity;

    @property (strong, nonatomic) UIButton *avatar;
    @property (strong, nonatomic) UILabel *nicknameLable;
    @property (strong, nonatomic) UIImageView *sexImg;
    @property (strong, nonatomic) UIImageView *timeIndicatorImg;
    @property (strong, nonatomic) UILabel *timeLable;
    @property (strong, nonatomic) UILabel *contentLable;
    @property (strong, nonatomic) UIImageView *contentImg;
    @property (strong, nonatomic) UILabel *jiongLable;
    @property (strong, nonatomic) UILabel *commentLable;
    @property (strong, nonatomic) UIButton *thumbUpBtn;
    @property (strong, nonatomic) UIButton *thumbDownBtn;
    @property (strong, nonatomic) UIButton *commentBtn;
    @property (strong, nonatomic) UIButton *shareBtn;

    -(void)removeConstraints;
    - (void)setEntity:(Articles *)entity;
    @end

把界面要显示的控件属性写上，还有模型entity

### 2.JJTableViewCellPic.m文件部分

>*写一个标志didSetupConstraints

记录cell是否添加了约束，防止重复添加，造成约束重复的冲突

    @interface JJTableViewCellPic()
    @property (assign,nonatomic) BOOL didSetupConstraints;
    @end

override 系统函数 updateConstraints

    - (void)updateConstraints {
        if (!self.didSetupConstraints) {
            [self setUpConstraints];
        }
       [super updateConstraints];
    }

>*override updateCellConstraints 和 layoutSubviews

在写完自定义的约束后需要主动调用updateCellConstraints去更新约束

    - (void)updateCellConstraints {
    [self setNeedsUpdateConstraints];
    [self updateConstraintsIfNeeded];
    
    [self setNeedsLayout];
    [self layoutIfNeeded];
    }

    - (void)layoutSubviews
    {
    [super layoutSubviews];
    [self.contentView setNeedsLayout];
    [self.contentView layoutIfNeeded];
    self.contentLable.preferredMaxLayoutWidth = CGRectGetWidth(self.contentLable.frame);
    }

>* 添加控件约束

添加的约束需要从上至下有关系，不能跳过，不能系统没办法计算高度
对图片左右约束也要注意，写了图片width和height的约束，就只能设置图片到左边or图片到右边的inset，不能两个都设置，否者会冲突。
对图片也可以不约束width和height用设置左右两边的inset，右边设置relation为`NSLayoutRelationGreaterThanOrEqual` ,保证排版的美观，同时也可以让图片只适应大小

    if (self.cell_type==CellTypeText ) {
        [self.jiongLable autoPinEdge:ALEdgeTop toEdge:ALEdgeBottom ofView:_contentLable withOffset:20];
    }else{
     
         self.contentImg.contentMode=UIViewContentModeScaleToFill;
        self.contentImg.clipsToBounds = YES;
        [self.contentImg autoPinEdge:ALEdgeTop toEdge:ALEdgeBottom ofView:_contentLable withOffset:10];
        
        [self.contentImg autoPinEdge:ALEdgeLeading toEdge:ALEdgeLeading ofView:_avatar];
    
      //[self.contentImg autoPinEdgeToSuperviewEdge:ALEdgeTrailing withInset:10];
         [self.contentImg autoPinEdgeToSuperviewEdge:ALEdgeTrailing withInset:10 relation:NSLayoutRelationGreaterThanOrEqual];
     
     //CGFloat width=K_SCREEN_WIDTH-20;
     //CGFloat height= (self.image_size.height/ self.image_size.width)*width;
        
      /**[self.contentImg autoSetDimension:ALDimensionWidth toSize:width];
      [self.contentImg autoSetDimension:ALDimensionHeight toSize:height];
    */
       [self.jiongLable autoPinEdge:ALEdgeTop toEdge:ALEdgeBottom ofView:_contentImg withOffset:20];
    }
    
    [self.jiongLable autoPinEdge:ALEdgeLeading toEdge:ALEdgeLeading ofView:_avatar];

>* cell里面的图片占位问题

从网络下载的图片可以从接口里面预先知道size，然后用一个大图裁剪出这个size放上去占着，这样裁剪的或许可以缓存起来，目前还没想到怎么优化这个占位图片。
网络图片一定要用sdImage这种插件类似的，没有缓存的cell会很卡，cell复用可能会图片错位

>* cell的复用问题

主要问题是约束冲突问题，假如你直接从队列里面拿回来的cell进行设置模型，更换数据，非常容易冲突，某些情况下可能不会。
为了防止冲突，首先要去掉上一次加在contentView上的所有约束，假如你还设置了其他约束不是加到父控件的，而是加在本身得，比如图片的宽度和高度，那么也得专门去掉

假如你是用的xib可以跳过本条，xib貌似自动给去掉了还是怎么的，可以直接用

    -(void) removeConstraints{
     _didSetupConstraints = NO;
    
    [self.contentView.constraints autoRemoveConstraints];
    
     /**  [self.contentImg.constraints autoRemoveConstraints];*/ //加了width height约束时，必须有
    }

### 3.JJFirstViewController.m部分
>* 在`viewDidLoad`里面注册一下，在复用的时候就可以少些很多判断，直接`dequeueReusableCellWithIdentifier`出来的就能用，代码可读性加强

    [self.tableView registerClass:[JJTableViewCellPic class] forCellReuseIdentifier:cellIdentifierPic];

>* 缓存好计算过的cell高度

把高度缓存到模型里面，模型决定视图
    -(CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{

     Articles *article=[_article_arr objectAtIndex:indexPath.row];
   
    if (article.cell_height) return article.cell_height;
    
    if (!self.propertyCellPic)   self.propertyCellPic = [self.tableView dequeueReusableCellWithIdentifier:cellIdentifierPic];
    
    [self configureCellPic:self.propertyCellPic atIndexPath:indexPath];
    
    CGSize size= [self.propertyCellPic.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    
    article.cell_height=size.height+1;//高度应该缓存到模型里
    
    return  article.cell_height;

    }

计算高度用的是`systemLayoutSizeFittingSize:`函数



###结尾

代码因为是练手的，PHP后端接口没有上传，目前其他人不能运行demo，可以自己改一下数据源来用

效果1:
![效果1](/static/img/444.gif)

效果2:
![效果2](/static/img/555.gif)

[github地址](https://github.com/jeffdeng/jiongjiongyoushen)





