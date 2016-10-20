[TOC]
	
    基于android6.0 源码分析
		
		前提：
		ipc通信机制bindler
    
#tcl项目完成的开发事项
  1. 最近应用的清除
  2. 单手模式的开发
  3. 通知自定义背景颜色的加入
  4. 快捷项的加如
  a. waves
  b. 单手模式
  c.静音时通知的弹出

##1. 通知机制的介绍
 * 通知demo（感性认识）
 * 通知notification.java类详解
	1. 路径：frameworks/base/core/java/android/app/Notification.java
	2. 作用：封装了所有通知的信息，使用见通知demo和doc文档
	3. //todo 通知是怎么传递到systeui的以显示详解
	4. //类的组织原理（类部类的机制）
 * 启动流程大致原理
	1. 相关源码的位置

	```java
		 framework/base/core/java/android/app/NotificationManager.java
		 framework/base/core/java/com/android/internal/statusbar/StatusBarNotification  implements Parcelable
		 framework/base/core/java/com/android/internal/statusbar/IStatusBar.aidl
		 framework/base/core/java/com/android/internal/statusbar/IStatusBarService.aidl
		 framework/base/core/java/com/android/internal/statusbar/StatusBarNotification.aidl   

		 framework/base/services/java/com/android/server/NotificationManagerService.java{@hide} extends INotificationManager.Stub
		 framework/base/services/java/com/android/server/StatusBarManagerService.java  extends IStatusBarService.Stub

		 framework/base/packages/SystemUI/src/com/android/systemui/statusbar/StatusBarService.java extends Service implements CommandQueue.Callbacks
		 framework/base/packages/SystemUI/src/com/android/systemui/statusbar/CommandQueue.java extends IStatusBar.Stub
	```
	
	 2. 流程
	 		 启动
			 (启动的流程)man（）->run()->startOtherServices()
			 1. 系统启动的时候：framework/base/services/java/com/android/server/SystemServer.java中：
				```java
				..........
							if (!disableSystemUI) {
									try {
											Slog.i(TAG, "Status Bar");
											statusBar = new StatusBarManagerService(context, wm);
											ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar); //启动StatusBarManagerService
									} catch (Throwable e) {
											reportWtf("starting StatusBarManagerService", e);
									}
							}
				.......
				.......
						
            mSystemServiceManager.startService(NotificationManagerService.class);       //启动 NotificationManagerService
            notification = INotificationManager.Stub.asInterface(
                    ServiceManager.getService(Context.NOTIFICATION_SERVICE));
            networkPolicy.bindNotificationManager(notification);

            mSystemServiceManager.startService(DeviceStorageMonitorService.class);
				......
				```
			这段代码是注册状态栏管理和通知管理这两个服务。
			 2. 在StatusBarManagerService.java中，
			 有addNotification（），
			 removeNotification（）,
			 updateNotification（）
			 等方法用于管理传递给他的通知对象。
				  这个类是一些管理方法，实际执行相关动作的是在IStatusBar.java里面，这个是framework/base/core/java/com/android/internal/statusbar/IStatusBar.aidl自动生成的用于IPC的类。
				
				```java
					public IBinder addNotification(StatusBarNotification notification) {   
											synchronized (mNotifications) {   
											IBinder key = new Binder();   
											mNotifications.put(key, notification);   
											if (mBar != null) {               //mBar其实就是IStatusBar的实例
											try {   
													mBar.addNotification(key, notification);   
												} catch (RemoteException ex) {   
										 }   
									}   
									return key;   
							}   
						}  
				```
		     为了防止NPE，每次使用mBar都先判断是否为null，mBar是在方法registerStatusBar中传递进来的。
		
				```java

								public void registerStatusBar(IStatusBar bar, StatusBarIconList iconList,   
												List<IBinder> notificationKeys, List<StatusBarNotification> notifications) {   
										
										enforceStatusBarService();   

										Slog.i(TAG, "registerStatusBar bar=" + bar);   
										mBar = bar;   
										synchronized (mIcons) {   
												iconList.copyFrom(mIcons);   
										}   
										synchronized (mNotifications) {   
												for (Map.Entry<IBinder,StatusBarNotification> e: mNotifications.entrySet()) {   
														notificationKeys.add(e.getKey());   
														notifications.add(e.getValue());   
												}   
										}   
										}

				```
		     framework/base/packages/SystemUI/src/com/android/systemui/statusbar/CommandQueue.java实现IStatusBar.java接口，
				 framework/base/packages/SystemUI/src/com/android/systemui/statusbar/StatusBarService.java提供IStatusBar相关服务。

				 CommandQueue.java中，IStatusBar.java里面对应的方法是用callback的形式调用的，callback的实现当然就在对应的服务提供类也就是StatusBarService.java中提供的啦。
				 CommandQueue.java中：
				```java
							public void addNotification(IBinder key, StatusBarNotification notification) {   
							 synchronized (mList) {   
									NotificationQueueEntry ne = new NotificationQueueEntry();   
									ne.key = key;   
									ne.notification = notification;   
									mHandler.obtainMessage(MSG_ADD_NOTIFICATION, 0, 0, ne).sendToTarget();   
											//这句话对应的mHandler执行语句是：   
											//  final NotificationQueueEntry ne = (NotificationQueueEntry)msg.obj;   
									// mCallbacks.addNotification(ne.key, ne.notification);   
											//也就是调用回调函数里面的addNotification。   
								}   
						}
				```
				在StatusBarService.java中：
				```java 
				    		mCommandQueue = new CommandQueue(this, iconList);   //StatusBarService实现了CommandQueue中的CommandQueue.Callbacks接口   
								mBarService = IStatusBarService.Stub.asInterface(ServiceManager.getService(Context.STATUS_BAR_SERVICE));   
								try {   
												//将IStatusBar实现类的对象传递到StatusBarManagerService.java中，这里的mCommandQueue就是上面对应的mBar啦。   
										mBarService.registerStatusBar(mCommandQueue, iconList, notificationKeys, notifications);   
								} catch (RemoteException ex) {   
										// If the system process isn't there we're doomed anyway.   
								}
				```
				最终执行状态栏更新通知等事件都是在实现的CommandQueue.Callbacks里面执行。还是以addNotification为例：
				```java
					public void addNotification(IBinder key, StatusBarNotification notification) {   
					boolean shouldTick = true;   
					if (notification.notification.fullScreenIntent != null) {   
							shouldTick = false;   
							Slog.d(TAG, "Notification has fullScreenIntent; sending fullScreenIntent");   
							try {   
									notification.notification.fullScreenIntent.send();   
							} catch (PendingIntent.CanceledException e) {   
							}   
					}    

					StatusBarIconView iconView = addNotificationViews(key, notification);   
					if (iconView == null) return;   
						//。。。以下省略N字。
				```
				大致流程就是:
				**调用StatusBarManagerService.java中的addNotification方法
				->（mBar不为空的话）执行mBar.addNotification(key, notification);
				->对应的是CommandQueue中的addNotification(IBinder key, StatusBarNotification notification)
				->CommandQueue中的mCallbacks.addNotification(ne.key, ne.notification);
				->StatusBarService中的addNotification。**
				
				3. 上面是提供相关功能的一些类，具体的notification的管理类是framework/base/services/java/com/android/server/NotificationManagerService.java，
				从该类的定义public class NotificationManagerService extends INotificationManager.Stub可以知道他是用来实现接口中INotificationManager中定义的相关方法并向外部提供服务的类。
				主要向外提供public void enqueueNotificationWithTag(String pkg, String tag, int id, Notification notification,int[] idOut)方法。该方法实际上是调用public void enqueueNotificationInternal(String pkg, int callingUid, int callingPid,String tag, int id, Notification notification, int[] idOut)，他里面提供了notification的具体处理方法。

 				摘取部分代码片段看看：
 				```java
				if (notification.icon != 0) {   
                StatusBarNotification n = new StatusBarNotification(pkg, id, tag,   
                        r.uid, r.initialPid, notification);   
                if (old != null && old.statusBarKey != null) {   
                    r.statusBarKey = old.statusBarKey;   
                    long identity = Binder.clearCallingIdentity();   
                    try {   
                        mStatusBar.updateNotification(r.statusBarKey, n);   
                    }   
                    finally {   
                        Binder.restoreCallingIdentity(identity);   
                    }   
                } else {   
                    //省略。。。
					```
					当判断好需要更新通知的时候调用mStatusBar.updateNotification(r.statusBarKey, n);方法，这个就是StatusBarManagerService.java中的addNotification方法，这样就进入上面所说的处理流程了。
					4.  在3中的NotificationManagerService.java是管理notification的服务，服务嘛就是用来调用的，调用他的就是大家熟悉的NotificationManager了。
				  在NotificationManager.java中，有一个隐藏方法，用来得到INotificationManager接口对应的服务提供类，也就是NotificationManagerService了。
					```java
					/** @hide */  
					static public INotificationManager getService()   
					{   
							if (sService != null) {   
									return sService;   
							}   
							IBinder b = ServiceManager.getService("notification");   
							sService = INotificationManager.Stub.asInterface(b);   
							return sService;   
					}
					```
					再看看更熟悉的notify方法，其实是执行：
					  ervice.enqueueNotificationWithTag(pkg, tag, id, notification, idOut);也就是3中提到的那个对外公开的服务方法了，这样就进入了上面提到的处理流程了。
					```java
								public void notify(String tag, int id, Notification notification)   
					{   
							int[] idOut = new int[1];   
							INotificationManager service = getService();   
							String pkg = mContext.getPackageName();   
							if (localLOGV) Log.v(TAG, pkg + ": notify(" + id + ", " + notification + ")");   
							try {   
									service.enqueueNotificationWithTag(pkg, tag, id, notification, idOut);   
									if (id != idOut[0]) {   
											Log.w(TAG, "notify: id corrupted: sent " + id + ", got back " + idOut[0]);   
									}   
							} catch (RemoteException e) {   
							}   
					}
		```



##2. systemui的介绍

			1)状态栏信息显示，比如电池，wifi信号，3G/4G等icon显示
			2)通知面板，比如系统消息，第三方应用消息，都是在通知面板显示。
			3)近期任务栏显示面板。比如长按主页或近期任务快捷键，可以显示近期使用的应用。
			4)提供截图服务。比如电源+音量加可以截图。
			5)提供壁纸服务。比如壁纸的显示。
			6)提供屏保服务。
			7)系统UI显示。比如系统事件到来时，显示系统UI提示用户。

1. AndroidMani.xml和android.mk 文件
2. systemui的代码目录结构和系统的整体架构
3. systemui的启动流程
4. 如何实现最近应用列表的清除功能
5. 其他

##3. 通知的机制以及systemui的联系
* systemui是如何管理通知
1.systemu如何显示通知，如何分类，如何解析的
2.动画的实现
3.交互：点击滑动后做了些什么
* 
##4. 通知的调试
1.怎样确认通知的属于那个应用
2.logcat 怎么看
##5.常见的问题

