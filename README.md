## Android-IM架构设计
### 1. 架构总览
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E6%9E%B6%E6%9E%84%E6%80%BB%E8%A7%88.jpg)
### 2. 模块介绍
#### 2.1 协议封装与任务流程
##### 1) 协议与任务的封装
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E5%8D%8F%E8%AE%AE%E4%B8%8E%E4%BB%BB%E5%8A%A1%E7%9A%84%E5%B0%81%E8%A3%85.jpg)  
 a. 协议有协议头(协议头因为格式相同，被抽象出来)和协议体组成，协议有两类：请求协议(request)和回复协议(response)；
 b. 任务(action)由请求协议、回复协议和任务回调(callback)组成；
 c. callback是针对客户端主动请求协议的相应处理，分别是成功回调、超时回调和失败回调；

##### 2) 消息(任务)流程
![](http://mmbiz.qpic.cn/mmbiz/sXiaukvjR0RDDvG75QJeex8eic87gwYb49ySyteIhoNMyeZzMictbWr135ibWTzlMYOqf8F2qicJlTqSNXZoucsWZZg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)  
a. 由UI或SYSTEM触发一个消息的生成，随之将其投递到发送队列中，等待发送；
b. 消息发送线程会不停的从发送队列中拉取一个消息并发送出去，同时放入超时监测队列；而当网络断开时会等待若干时间并重新循环，若检测该消息在队列等待时间过长，则丢弃该消息并触发相应的失败回调；
d. 对于客户端主动请求，在收到服务端给予回复时，调用该消息的成功回调处理相应回复；
e. 对于超时线程监测超时的消息，移除超时监测队列，并调用超时回调。

#### 2.2 定时任务
定时任务的实现主要由TimerHelper类与ITimerProcessor两个类，第一个类主要实现了Timer的功能，ITimerProcessor为一个接口，当要建立一个Timer的时候，需要新建立一个类实现ITimerProcessor中得process接口来处理具体的业务。

TimerHelper类的内部封装了系统的Timer及TimerTask来实现，对外提供startTimer以及stopTimer接口。startTimer需要一个布尔类型的参数标志启动定时任务是需要执行一次，还是永久执行。

新生成TimerHelper实例的时候需要制定定时任务的时间以及定时触发处理接口ITimerProcessor的具体实现。

当然在Timer的设计中，我们并没有完全采用系统提供的Timer类实现，有些Timer是我们采用线程模拟实现，这个主要是基于Java 中Timer的一些缺陷来考虑。下面简单介绍下四种Timer

##### 2.2.1 心跳Timer
心跳Timer 是维持客户端与服务端长连一个强有力的保证。网络中接收发送都是使用socket的recv与send进行发送与接收，如果此套接字已经断开，则发送与接收数据都会出现问题，创建心跳机制，就是为了及时检测该套接字是否有效。

所谓心跳就是给服务端发送一个自定义的包，来告诉服务端，自己在线，以确保长连的有效性。

##### 2.2.2 发送超时Timer
在开发网络应用程序的时候，处理业务和通讯流程之间经常会出现矛盾。这种矛盾主要是由于两者之间的不同步造成的。比如，网络的延迟较大，而实际业务处理的速度则相对比较快，那么如果处理完某一事务然后等待发送成功再处理下一个事务则会大大降低效率。所以我们建立了一个发送消息队列。这是一个典型的“生产者消费者”模型，业务逻辑将需要发送的数据放到消息队列中，SendPacketMonitor从消息队列中取出数据，并发送出去。

由于使用了消息队列的模式，发送就变成了一种异步操作，业务逻辑将消息放入消息队列后，就可以进行其他的操作，而无法知道该消息是否真正发送成功。因此，我们在设计消息队列的时候采用了回调机制，业务方在放入消息队列的时候，必须实现onSuccess接口与onTimeOut接口，分别在发送成功与发送超时调用。

发送超时Timer是自己采用线程来模拟实现的，在SendPacketMonitor类，作为消费者，会不停的从消息队列中取出数据，取出数据后，会判断该消息产生的时刻与当前时间相比较，如果发现时差已经超过系统定义的最大超时时间，则直接调用“生产者”的onTimeOut接口，通知其发送超时。

##### 2.2.3 重连“Timer”
重连是另一个保证长连的机制，虽然我们使用了心跳机制来保证长连，但是由于网络环境的复杂性，我们无法保证在一个连接开启后，就永远保持连接，因此，重连就成了另外一个保证。

重连主要是为了当连接断开的时候，客户端能够自动快速的连接到服务端。为了系统的稳定性，及相应快速，重连Timer采用的是线程模拟Timer实现。

重连的逻辑中，会去检测服务端的心跳包，如果发现长时间没有收到服务端的任何数据包，则认为该socket已经失效，并进行重连。

在重连Timer中，为了防止雪崩效应的出现，我们在检测到socket失效，并不是立马进行重连，而是让客户端随机Sleep一段时间再去连接服务端，这样就可以使不同的客户端在服务端重启的时候不会同时去连接，从而造成雪崩效应。

##### 2.2.4 好友状态Timer
虽然在实现的逻辑中，服务端在好友状态变化的时候，会主动推送消息给客户端，但是我们还是设计了好友状态Timer。因为在网络复杂的环境中，有太多的未知因素。

好友状态Timer的基本实现就是每隔一段时间发送一个数据包请求获得好友的状态信息。当收到响应数据包的时候，就会去更新Cache中的好友状态信息。

#### 2.3状态管理
##### 2.3.1 状态管理StateManager
状态管理就是一个多状态的状态机，其中包括Net状态管理(指硬件网络是否可用)，Socket状态管理(指与MsgServer的连接是否可用)。

状态管理主要功能就是采集Net状态与Socket状态，提供两个接口notifyNetState与notifySocketState两个接口供Net状态管理与Socket状态管理调用，当接收到状态变化的时候会调用NetDispach进行网络状态变更分发。

##### 2.3.2 Net状态管理NetStateManager
Net状态管理指的是物理网络状态的管理，主要管理当前物理网络是否可用以及进行变化时进行监听，当监听到网络断开或者连上事件的时候，会调用状态管理notifyNetState接口。
在应用启动的时候我们会注册网络变化广播接收
public class ConnectionChangeReceiver extends BroadcastReceiver
Override他的onReceive函数，在onReceive 函数中获取网络连接服务，然后调用NetStateManager的setState接口，通知状态变化。

##### 2.3.3 Socket状态管理
Socket状态管理指的是客户端与MsgServer之间的连接状态。当检测到连接不可用时，会调用该接口的setState接口设置状态。当socket的channelConnected、exceptionCaught与channelDisconnected函数被调用的时候以及在重连出现异常或失败的时候会通知进行状态变更。

##### 2.3.4 状态变更分发
状态变更分发对外提供三个接口register，unregister与dispachMsg接口。外界如果关心网络状态变化事件，可以注册自己的Handler到该类，当网络状态发生变化的时候，会根据注册的Handler进行事件通知。

register接口为提供注册的接口。

unregister接口为取消注册的忌口。

dispachMsg为事件分发接口，当网络状态发生变化的时候，该接口会被调用，通知各个Handler进行处理。

#### 2.4 断线重连
断线重连机制是当IM与MsgServer断开后能够自动连接。断线重连为一个单独的线程，进行循环，当检测到NetState为不可用的时候，会随机睡眠1-9秒然后继续检测。同时会检测心跳包，当发现最后一次收到心跳包超过MAX_HEART_BEAT_TIME时间会认为Socket连接不可用而重置SocketState的状态。当检测到网络可用，Socket状态不可用的时候，就会启动断线重连机制，在进行断线重连之前，会进行连接次数判断，如果为第一次重连，则随机睡眠1秒多，这个机制主要是为了防止服务端出现异常而重启的时候大量的客户端同时连接上来而发生雪崩现象。如果不为第一次重连，则睡眠指定时间，该时间的计算公式如下：
nSleep = (long) Math.pow(2, mnReconnectCount);
if(nSleep > 16) {
nSleep = 16;
}
该计算公式主要是为了防止大量的客户端不停的进行重连从而对服务端造成大量的压力，另外从节省客户端的能耗考虑。

每次进行重连都会将重连次数累加，这个主要是为了防止以后需要对重连次数进行限制。
重连过程如下图：
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E6%96%AD%E7%BA%BF%E9%87%8D%E8%BF%9E.jpg)

#### 2.5 登陆
登陆流程主要分为以下几个流程：1、认证；2、获取MsgServer地址；3、登陆MsgServer。下面依次介绍。

##### 2.5.1 认证
认证过程主要是对用户合法身份的验证，包括如下两个方面：
从主客获取AppToken。该过程是一个反射调用，IM程序调用主客的获取Token接口获取到主客的AppToken，
Dao等信息。

拿从主客获取到得AppToken及Dao信息到IM的验证服务器去换取一个IMToken，该过程为一个Http调用。服务器会对上传的AppToken及Dao信息进行校验，如果校验成功，则会返回一个IMToken，以及LoginServer地址等信息，后期需要拿该Token到MsgServer进行登陆验证。

认证的过程主要在TokenManager中实现，该类中还对Token时效进行了管理，当获取IMToken的时候会先对判断IMToken是否为空，如果为空则去获取AppToken等信息，再去服务端换取IMToken，否则判断Token的时效是否失效效，如果失效则获取AppToken并换取IMToken信息，如果有效，则直接返回IMToken。

##### 2.5.2 获取MsgServer地址
该过程是客户端通过验证时拿到的LoginServer地址建立Socket连接，并发送获取MsgServer请求，LoginServer会返回一个可用的MsgServer的Ip及Port。

##### 2.5.3 登陆MsgServer
当经过3.2获取到MsgServer的IP及Port后，会更具给定的IP与Port与MsgServer建立一个Socket连接，当连接建立成功后。会携带获取到得IMToken，用户名等信息发送一个登陆请求包，如果登陆请求验证通过，客户端启动一个Timer与服务端发送心跳包保持长连接。

以下为整个登陆的流程:
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E7%99%BB%E9%99%86MsgServer.jpg)

#### 2.6 异步实现
异步的封装有两种实现方式：1、需要更新界面的异步；2、不需要更新界面的异步。两种实现的方式是不同的。同时异步还有一个异步管理类。

##### 2.6.1 TaskTrigger类
TaskTrigger类主要用来注册以及管理各个Task，每生成一个异步实例，都需要通过trigger接口如果对于需要更新界面的异步，则直接调用AsyncTask的execute接口执行任务，否则将其放入Task的任务队列中，后台会通过process接口调用dotrigger接口执行具体的任务，执行结束后，调用task的callback接口。

##### 2.6.2 需要更新界面的异步
该方式主要集成Android自有的AsyncTask类，对其进行了一个简单的封装，该异步实现方式主要解决一些需要更新UI界面的异步，解决Android中飞UI线程不能更新UI的问题。

##### 2.6.3 不需要更新界面的异步
该异步主要提供不需要更新UI的一些异步操作，该实现方式为新开启一个线程执行任务，当任务执行完成之后调用回调函数。

#### 2.7 本地缓存
存消息时,根据缓存中之前一条消息的ID,获得新消息的唯一自增ID,将该消息存入DB。存消息时依赖三张表,联系人表,主消息表和附加消息表。首先会根据当前收发用户的ID从联系人表中得到两者的唯一联系ID,并设置消息中的联系ID,将该消息存入主消息表和附加消息表。  
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E6%9C%AC%E5%9C%B0%E7%BC%93%E5%AD%98.jpg)  
读消息时,直接从DB中拉取即可,根据参数的不同拉取的消息不同。可以根据消息ID,拉取某一天消息,也可以根据起止消息ID,拉取指定偏移量和指定条数的消息列表。  
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E8%AF%BB%E6%B6%88%E6%81%AF.jpg)  
存用户信息时,判断该用户信息是否为空或合法,若为空或不合法,直接返回;若合法,则更新 缓存中的用户信息,用户信息只保留在缓存中。  
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E5%AD%98%E7%94%A8%E6%88%B7%E4%BF%A1%E6%81%AF%E6%97%B6.jpg)  
取用户信息时,先判断缓存中是否存在该用户信息,若存在,则直接返回该用户信息;否则返回null，由业务端选择是否发起取用户信息的操作。  
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E5%8F%96%E7%94%A8%E6%88%B7%E4%BF%A1%E6%81%AF%E6%97%B6.jpg)  
收发消息时,得到好友ID,判断缓存中得最近联系人ID列表中是否存在,若不存在则添加,否则 忽略;同样地,在获取最近联系人列表时也如此。  
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E6%94%B6%E5%8F%91%E6%B6%88%E6%81%AF%E6%97%B6.jpg)  
⾸首先获得最近联系人ID列表,然后从缓存中读取必要地信息,组合成最近联系人列表。另外,启用一个线程,每500毫秒来检测是否有新消息来通知UI主线程来更新界面  
![](http://7xufmk.com1.z0.glb.clouddn.com/im%E8%8E%B7%E5%BE%97%E6%9C%80%E8%BF%91%E8%81%94%E7%B3%BB%E4%BA%BAID.jpg)
