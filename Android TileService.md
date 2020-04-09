# Android TileService

## TileService简介

Tileservice为用户提供了可添加到QuickSettings的快捷开关按钮，Systemui中提供了一块区域给QS[^1]，开发者可以通过TileService实现可添加到systemUI的快捷开关，用户可以通过开关实现一些操作而不用进入app中。TileService的设计结构主要分为两部分：查询系统中所有TileService信息；对TileService进行绑定操作以及生命周期管理。以下说明文档基于Android 28。

## TileService相关数据管理模块

这一模块中核心类是QSTileHost,此类由Statusbar类通过SystemUIFactory创建并传递给QS。它存储当前已填加的QSTile[^2]列表，并提供增删查改操作，每个被添加的QSTile会存储到系统数据库中，对应key为sysui_qs_tiles。同时QSTileHost监听到数据库的变化后更新列表。他还负责创建QSTile；创建QSTileView[^3]。
### QSTileHost的加载

QSTileHost在onTuningChanged回调中监听到QSTile相关数据的变化，在方法的回调参数获取Tile名称，如果为空则读取R.string.quick_settings_tiles文件中的当前已添加Tile信息。这其中有一条规则是如果存在名称为default的Tile则使用R.string.quick_settings_tiles_default文件中的默认Tile信息。已添加Tile信息存储方式是将所有名称拼接成一个字符串，中间用逗号隔开。
获取所有Tile信息列表后，QSTileHost会遍历列表创建Tile，Tile的基类实现是QSTileImpl，其中有一些很重要的基础操作。Tile创建的具体工作是QSFactoryImpl完成，系统自带的Tile都有相对应的Tile实现类，开发者创建的Tile实现类为CustomTile。

QSTileHost的创建时序图：
[![3EElA1.png](https://s2.ax1x.com/2020/02/19/3EElA1.png)](https://imgchr.com/i/3EElA1)

### 所有可添加的QSTile的查询流程

QS在初始化过程中创建TileQueryHelper，并由这个类执行QSTile的查询工作，查询分为系统自带的和其他包提供的QSTile。查询完成后回调通知QS列表adapter更新。

```java
 public void queryTiles(QSTileHost host) {
        mTiles.clear();
        mSpecs.clear();
        mFinished = false;
        // Enqueue jobs to fetch every system tile and then ever package tile.
        addStockTiles(host);
        addPackageTiles(host);
    }
```
TileQueryHelper查询流程时序图：
![3EE3h6.png](https://s2.ax1x.com/2020/02/19/3EE3h6.png)


## 对TileService的管理

TileSerivce框架的基础原理是SystemUI对TileSerivce绑定后使用aidl方式跨进程通信，SystemUI在发起绑定的同时也会将自身的binder传给TileService，这样通信的双方都可以调用对方的方法。TileService返回的binder对应的是IQSTileService接口，在SystemUI中的TileServices类，实现IQSService接口，并将binder传递给TileService。

### 几个重要类详细说明：
TileServices:这个类不是Android Service组件，这个类实例与QSTileHost一同创建，在程序生命周期内只有一个实例，这个类实现了aidl接口IQSService，因此它负责处理所有TileService对SystemUI的调用请求。内部维护3个表mServices，mTiles，mTokenMap。

TileServiceManager:每个QSTile被创建后，QSTile会调用TileServices的getTileWrapper方法，该方法为每个QSTile创建分配相对应的TileServiceManager，这个类的功能是管理QSTile的Service的连接状态。

TileLifecycleManager:功能是管理对应的TileService的连接生命周期，向TileServiceManager发起的绑定/解绑请求最终都会由这个类处理，是最终发起bindService和unbindService的地方，持有TileService的binder对象，可以最终调用aidl接口方法，同时处理连接断开。

### onStartListening
onStartListening流程的目的在于通知TileService，SystemUI开始监听TileService的状态变化同时也会刷新QSTile的状态。QS在下拉后会下发onStartListening指令，该指令经过层层下发到达QSTile，每个QSTile内部维护一个相对应的TileServiceManager,在这个manager中判断此TileService是否已绑定，如果未绑定则委托TileLifecycleManager发起service绑定操作，最后再由TileLifecycleManager调用TileService的onStartListening方法。

### onStopListening
onStopListening的流程和onStartListening相似，目的在于通知TileService，SystemUI不再监听来自它的状态变化，也就是即使TileService修改Tile[^4]的状态，SystemUI也不会立即刷新而是在下一次onStartListening时主动获取状态并刷新。QS在收起后会下发onStopListening指令，该指令经过层层下发到达QSTile，由TileLifecycleManager调用TileService的onStopListening方法，然后向TileServiceManager发起解绑请求，默认30s后解除绑定。

CustomTile.java处理监听实现如下：

```java
@Override
    public void handleSetListening(boolean listening) {
        if (mListening == listening) return;
        mListening = listening;
        try {
            if (listening) {
                setTileIcon();
                refreshState();
                if (!mServiceManager.isActiveTile()) {
                    mServiceManager.setBindRequested(true);
                    mService.onStartListening();
                }
            } else {
                mService.onStopListening();
                if (mIsTokenGranted && !mIsShowingDialog) {
                    try {
                        if (DEBUG) Log.d(TAG, "Removing token");
                        mWindowManager.removeWindowToken(mToken, DEFAULT_DISPLAY);
                    } catch (RemoteException e) {
                    }
                    mIsTokenGranted = false;
                }
                mIsShowingDialog = false;
                mServiceManager.setBindRequested(false);
            }
        } catch (RemoteException e) {
            // Called through wrapper, won't happen here.
        }
    }
```

TileServiceManager.java处理绑定请求实现如下

```java
public void setBindRequested(boolean bindRequested) {
        if (mBindRequested == bindRequested) return;
        mBindRequested = bindRequested;
        if (mBindAllowed && mBindRequested && !mBound) {
            mHandler.removeCallbacks(mUnbind);
            bindService();
        } else {
            mServices.recalculateBindAllowance();
        }
        if (mBound && !mBindRequested) {
            mHandler.postDelayed(mUnbind, UNBIND_DELAY);
        }
    }
```
TileService的开始监听流程时序图

![3EE1tx.png](https://s2.ax1x.com/2020/02/19/3EE1tx.png)


## QS在Statusbar的视图结构中的位置

statusbar的根布局是super_status_bar.xml,其中包含两大块：1.固定在屏幕上方的状态栏，布局为status_bar.xml;2.可展开的负一屏，其中又主要包括QS和通知中心，布局文件为status_bar_expanded. 
QS中与TileService相关的两块布局是QSPanel和QSCustomizer。QSPanel包含正常状态下Tile，QSCustomizer是编辑Tile状态下的布局。




[^1]: SystemUI中的QuickSetting模块，核心类是QSFragment。 
[^2]: 代表快捷开关
[^3]: 代表快捷开关的view
[^4]: 代表快捷开关，与QSTile不同的是，QSTile用于SystemUI内部维护快捷开关，同时类中实现了很多控制逻辑。Tile用于进程间传输，结构比较简单，目的是为了SystemUI和TileService交换开关的视图状态。