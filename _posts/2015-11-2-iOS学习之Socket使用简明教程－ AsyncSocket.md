---
layout: post
title: "iOS学习之Socket使用简明教程－ AsyncSocket"
description: "iOS学习之Socket使用简明教程－ AsyncSocket"
category : lessons
tags: []
---




#iOS学习之Socket使用简明教程－ AsyncSocket

如果需要在项目中像QQ微信一样做到即时通讯，必须使用socket通讯，本人也是刚学习，分享一下，有什么不对的地方希望大家指正

ios原生的socket用起来不是很直观，所以我用的是AsyncSocket这个第三方库，对socket的封装比较好，只是好像没有带外传输（out—of-band） 如果你的服务器需要发送带外数据，可能得想下别的办法

环境

下载AsyncSockethttps://github.com/robbiehanson/CocoaAsyncSocket类库，将RunLoop文件夹下的AsyncSocket.h, AsyncSocket.m, AsyncUdpSocket.h, AsyncUdpSocket.m 文件拷贝到自己的project中

添加CFNetwork.framework, 在使用socket的文件头

	 #import <sys/socket.h>
	 #import <netinet/in.h>
	 #import <arpa/inet.h>
	 #import <unistd.h>
使用

1. socket 连接

即时通讯最大的特点就是实时性，基本感觉不到延时或是掉线，所以必须对socket的连接进行监视与检测，在断线时进行重新连接，如果用户退出登录，要将socket手动关闭，否则对服务器会造成一定的负荷。

一般来说，一个用户（对于ios来说也就是我们的项目中）只能有一个正在连接的socket，所以这个socket变量必须是全局的，这里可以考虑使用单例或是AppDelegate进行数据共享，本文使用单例。如果对一个已经连接的socket对象再次进行连接操作，会抛出异常（不可对已经连接的socket进行连接）程序崩溃，所以在连接socket之前要对socket对象的连接状态进行判断

使用socket进行即时通讯还有一个必须的操作，即对服务器发送心跳包，每隔一段时间对服务器发送长连接指令（指令不唯一，由服务器端指定，包括使用socket发送消息，发送的数据和格式都是由服务器指定），如果没有收到服务器的返回消息，AsyncSocket会得到失去连接的消息，我们可以在失去连接的回调方法里进行重新连接。

先创建一个单例，命名为Singleton

	  Singleton.h
	
	// Singleton.h
	 #import "AsyncSocket.h"

	 #define DEFINE_SHARED_INSTANCE_USING_BLOCK(block) \
	static dispatch_once_t onceToken = 0; \
	__strong static id sharedInstance = nil; \
	dispatch_once(&onceToken, ^{ \
	sharedInstance = block(); \
	}); \
	return sharedInstance; \
	
	@interface Singleton : NSObject
	
	+ (Singleton *)sharedInstance;
	
	@end
	Singleton.m
	
	+(Singleton *) sharedInstance
	{
	
	static Singleton *sharedInstace = nil;
	static dispatch_once_t onceToken;
	dispatch_once(&onceToken, ^{
	
	    sharedInstace = [[self alloc] init];
	});
	
	return sharedInstace;
	}
这样一个单例就创建好了

在.h文件中生命socket变量

	@property (nonatomic, strong) AsyncSocket    *socket;       // socket
	@property (nonatomic, copy  ) NSString       *socketHost;   // socket的Host
	@property (nonatomic, assign) UInt16         socketPort;    // socket的prot
下面是连接，心跳，失去连接后重连

连接(长连接)

	在.h文件中声明方法，并声明代理<AsyncSocketDelegate>
	
	-(void)socketConnectHost;// socket连接
	在.m中实现，连接时host与port都是由服务器指定，如果不是自己写的服务器，请与服务器端开发人员交流
	
	// socket连接
	-(void)socketConnectHost{
	
	    self.socket    = [[AsyncSocket alloc] initWithDelegate:self];
	
	    NSError *error = nil;
	
	    [self.socket connectToHost:self.socketHost onPort:self.socketPort withTimeout:3 error:&error];
	
	}
心跳

心跳通过计时器来实现 
在singleton.h中声明一个定时器

    @property (nonatomic, retain) NSTimer        *connectTimer; // 计时器
在.m中实现连接成功回调方法，并在此方法中初始化定时器，发送心跳在后文向服务器发送数据时说明

    #pragma mark  - 连接成功回调
    -(void)onSocket:(AsyncSocket *)sock didConnectToHost:(NSString  *)host port:(UInt16)port
{
    NSLog(@"socket连接成功");

    // 每隔30s像服务器发送心跳包
    self.connectTimer = [NSTimer scheduledTimerWithTimeInterval:30 target:self selector:@selector(longConnectToSocket) userInfo:nil repeats:YES];// 在longConnectToSocket方法中进行长连接需要向服务器发送的讯息

    [self.connectTimer fire];

}
2. socket 断开连接与重连

断开连接

失去连接有几种情况，服务器断开，用户主动cut，还可能有如QQ其他设备登录被掉线的情况，不管那种情况，我们都能收到socket回调方法返回给我们的讯息，如果是用户退出登录或是程序退出而需要手动cut，我们在cut前对socket的userData赋予一个值来标记为用户退出，这样我们可以在收到断开信息时判断究竟是什么原因导致的掉线

在.h文件中声明一个枚举类型
 
   
     enum{
    SocketOfflineByServer,// 服务器掉线，默认为0
    SocketOfflineByUser,  // 用户主动cut
    };
声明断开连接方法

	-(void)cutOffSocket; // 断开socket连接
.m

	// 切断socket
	-(void)cutOffSocket{

    self.socket.userData = SocketOfflineByUser;// 声明是由用户主动切断

    [self.connectTimer invalidate];

    [self.socket disconnect];
	}
重连

实现代理方法

	-(void)onSocketDidDisconnect:(AsyncSocket *)sock
	{
	    NSLog(@"sorry the connect is failure %ld",sock.userData);
	    if (sock.userData == SocketOfflineByServer) {
	        // 服务器掉线，重连
	        [self socketConnectHost];
	    }
	    else if (sock.userData == SocketOfflineByUser) {
	        // 如果由用户断开，不进行重连
	        return;
	    }
	
	}
3. socket 发送与接收数据

发送数据 
我们补充上文心跳连接未完成的方法

	// 心跳连接
	-(void)longConnectToSocket{
	
    // 根据服务器要求发送固定格式的数据，假设为指令@"longConnect"，但是一般不会是这么简单的指令

    NSString *longConnect = @"longConnect";

    NSData   *dataStream  = [longConnect dataUsingEncoding:NSUTF8StringEncoding];

    [self.socket writeData:dataStream withTimeout:1 tag:1];

}
socket发送数据是以栈的形式存放，所有数据放在一个栈中，存取时会出现粘包的现象，所以很多时候服务器在收发数据时是以先发送内容字节长度，再发送内容的形式，得到数据时也是先得到一个长度，再根据这个长度在栈中读取这个长度的字节流，如果是这种情况，发送数据时只需在发送内容前发送一个长度，发送方法与发送内容一样，假设长度为8

	NSData   *dataStream  = [@8 dataUsingEncoding:NSUTF8StringEncoding];
	
	[self.socket writeData:dataStream withTimeout:1 tag:1];
	接收数据 
	为了能时刻接收到socket的消息，我们在长连接方法中进行读取数据
	
	 [self.socket readDataWithTimeout:30 tag:0];
	如果得到数据，会调用回调方法
	
	-(void)onSocket:(AsyncSocket *)sock didReadData:(NSData *)data withTag:(long)tag
	{
    // 对得到的data值进行解析与转换即可

    [self.socket readDataWithTimeout:30 tag:0];

	}
4. 简单使用说明

我们在用户登录后的第一个界面进行socket的初始化连接操作，在得到数据后，将所需要显示的数据放在singleton中，对变量进行监听后做出相应的操作即可，延伸起来比较复杂，没有真实数据也不太方便说明，大家自己进行探索吧，有问题请在下方留言

    [Singleton sharedInstance].socketHost = @"192.186.100.21";// host设定
    [Singleton sharedInstance].socketPort = 10045;// port设定

    // 在连接前先进行手动断开
    [Singleton sharedInstance].socket.userData = SocketOfflineByUser;
    [[Singleton sharedInstance] cutOffSocket];

    // 确保断开后再连，如果对一个正处于连接状态的socket进行连接，会出现崩溃
    [Singleton sharedInstance].socket.userData = SocketOfflineByServer;
    [[Singleton sharedInstance] socketConnectHost];
    
    
    
    
    
 本文转自[这里](http://my.oschina.net/joanfen/blog/287238)本文涉及到的代码在[这里](http://www.oschina.net/code/snippet_735123_36974)