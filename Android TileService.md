# Android TileService

## TileService简介

Tileservice为用户提供了可添加到QuickSettings的快捷开关按钮，Systemui中提供了一块区域给QS，用户可以通过开关实现一些操作而不用进入app中。

## QS在Statusbar的视图结构中的位置

statusbar的根布局是super_status_bar.xml,其中包含两大块：1.固定在屏幕上方的状态栏，布局为status_bar.xml;2.可展开的负一屏，其中又主要包括QS和通知中心，布局文件为status_bar_expanded. 
QS中与TileService相关的两块布局是QSPanel和QSCustomizer。QSPanel包含正常状态下Tile，QSCustomizer是编辑Tile状态下的布局。

## QSTileHost创建分发过程

[![3EElA1.png](https://s2.ax1x.com/2020/02/19/3EElA1.png)](https://imgchr.com/i/3EElA1)

QSTileHost主要负责管理Tile的创建，销毁，加载工作。

## Tile的首次加载流程

QSTileHost在onTuningChanged回调中发起创建流程，在方法的回调参数获取Tile名称，如果为空则读取R.string.quick_settings_tiles文件中的当前已添加Tile信息。这其中有一条规则是如果存在名称为default的Tile则使用R.string.quick_settings_tiles_default文件中的默认Tile信息。已添加Tile信息存储方式是将所有名称拼接成一个字符串，中间用逗号隔开。

获取所有Tile信息列表后，QSTileHost会遍历列表创建Tile，Tile的基类实现是QSTileImpl，其中有一些很重要的基础操作。Tile创建的具体工作是QSFactoryImpl完成，系统自带的Tile都有相对应的Tile实现类，开发者创建的Tile实现类为CustomTile。

在Tile信息和实例都创建好后，UI模块即可通过Host获取Tile。

## QSCustomizer的查询Tile流程

![3EE3h6.png](https://s2.ax1x.com/2020/02/19/3EE3h6.png)

查询所有TileService的具体工作是TileQueryHelper完成的，方法是依赖PackageManager指定TileService独有的Action查询满足条件的service。

## TileService的绑定流程

![3EE1tx.png](https://s2.ax1x.com/2020/02/19/3EE1tx.png)

TileServices由TileHost创建，TileServices负责管理TileService的链接和断开，它可以控制TileService的连接数量以提高性能。
