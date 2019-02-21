# 演示 #
![Avater](gitImage/show.gif)

# MOBA 服务器 编写思路 #

## 编写思路 ##
### 底层问题 ###
#### 同步问题 ####
在游戏领域，同步可以大致分为下面这两种，即：

1. 状态同步
2. 帧同步

状态同步是将游戏中大部分需要计算的逻辑都放在服务器上进行计算.  

1. 优点是较为安全，可以防止游戏外挂的出现，如果有人打算制作外挂，必须先攻破游戏服务器
2. 缺点是性能问题，基于状态的同步，对服务器的性能是一个很大的挑战，且所有逻辑运算放到服务器上算，会导致游戏的延迟略高，对于实时性要求极高的游戏不适用。（即FPS，MOBA）

**我的理解**是帧同步是游戏内所有逻辑由客户端进行计算，只有在关键帧部分进行同步，所谓关键帧，类似于Flash中的说法，即在游戏中，某一帧是极为关键的，其他则只是作为过渡，举个例子，在一个联网的坦克大战游戏中，坦克发射炮弹的一瞬间就是关键帧，而坦克发射后，其运动轨迹并不需要服务器对其进行同步，而是由客户端进行计算，只有**坦克发射炮弹的那一瞬间**是需要同步的。

1. 优点：实时性强，服务端仅需要转发关键帧时的协议信息给各个客户端就可以了，对服务器的性能需求不算大
2. 缺点：安全性较弱，因为所有逻辑运算在客户端进行，容易被玩家更改本地数据从而制作游戏外挂

在本游戏中，因为是MOBA游戏，对游戏的实时性要求较高，故采用**帧同步**的方案。

#### 粘包、分包问题 ####
1.粘包产生原因

> 先说TCP：由于TCP协议本身的机制（面向连接的可靠地协议-三次握手机制）客户端与服务器会维持一个连接（Channel），数据在连接不断开的情况下，可以持续不断地将多个数据包发往服务器。
> 
> 但是如果发送的网络数据包太小，那么他本身会启用Nagle算法（可配置是否启用）对较小的数据包进行合并（基于此，TCP的网络延迟要UDP的高些）然后再发送（超时或者包大小足够。
> 
> 那么这样的话，服务器在接收到消息（数据流）的时候就无法区分哪些数据包是客户端自己分开发送的，这样产生了粘包；服务器在接收到数据库后，放到缓冲区中，如果消息没有被及时从缓存区取走，下次在取数据的时候可能就会出现一次取出多个数据包的情况，造成粘包现象（确切来讲，对于基于TCP协议的应用，不应用包来描述，而应用“流”来描述）。
> 
> 个人认为服务器接收端产生的粘包应该与linux内核处理socket的方式 select轮询机制的线性扫描频度无关。
> 
> 再说UDP：本身作为无连接的不可靠的传输协议（适合频繁发送较小的数据包），他不会对数据包进行合并发送（也就没有Nagle算法之说了），他直接是一端发送什么数据，直接就发出去了，既然他不会对数据合并，每一个数据包都是完整的（数据+UDP头+IP头等等发一次数据封装一次）也就没有粘包一说了。

2.分包产生原因

> 分包产生的原因：
> 
> 可能是IP分片传输导致的，也可能是传输过程中丢失部分包导致出现的半包，还有可能就是一个包可能被分成了两次传输，在取数据的时候，先取到了一部分（还可能与接收的缓冲区大小有关系），总之就是一个数据包被分成了多次接收。

**解决方案**

#### 心跳机制 ####

### 协议层问题（协议设计） ###

#### 登录功能 ####

登录--指的是一个玩家在Socket连接上服务器后，还没开始游戏的一种状态，在这种状态下，客户端不应该接受任何关于游戏逻辑的消息。

完成登录功能需要设计以下协议：

1. LoginConn 协议，参数如下：
	1. userName : string : 用户名(即用户ID)
	2. password : string : 密码

当玩家按下登录按钮后,向服务器发送Login消息.

2. LoginResultConn 协议,参数如下:
	1. userName : string : 用户名,用来标识唯一用户
	2. loginStatus : string : 登录状态,success,fail这两种

当服务器收到Login消息后,解析协议,得到用户名和密码后,对数据库进行搜索,根据数据库是否存在用户名和密码,构造LoginResult协议给用户.

客户端额外处理:  
当客户端接收到LoginStatus为Success的消息时,自动进入登录界面,即多人游戏--加入房间和创建房间的界面.

登录功能难点:

Login/LoginReuslt协议应该只在处于登录状态的用户之间传递,而不应该传递给那些已经登录完毕的单位.

在这里我的解决方法是,服务端使用一个ChannelGroup(记为LoginChannelGroup)来存储所有加入进来的连接(Channel),当一个用户登录完成后,LoginChannelGroup应该关闭这条连接(Channel),并将这条连接加入到另一个ChannelGroup中(此处记为RoomChannelGroup).

另一个难点,我使用反射的方法来对消息进行分发,同时,HandlerConn用于处理玩家未登录前的状态,HanlerPlayer用于处理玩家登录后的游戏逻辑处理.问题在于,我应该如何判断某个消息应该分发到HandlerConn中还是HanlerPlayer.

我采用的解决方法是,对协议名进行限定,即所有处理玩家未登录前状态的协议,其名称均会有Conn这几个字.


#### 房间系统 ####

房间系统指的是，玩家在登录状态完成后，将会进入多人游戏大厅，在游戏大厅中，用户可以加入其它人的房间进行游玩，或者自己创建房间等待其它玩家。  
房间系统的特别之处在于在同一房间内的玩家发送的信息,只会被服务器转发给同一房间的玩家,其他玩家(处于登录状态/处于其他房间)都收不到在这个房间内被互相传递的信息.

完成登录功能需要设计以下协议：

1. CreateRoom 创建房间协议,用户点击创建房间按钮,向服务器发送创建房间的信息,参数如下:
	1. userName : string : 创建房间的用户名
	2. roomName : string : 创建的房间的名字

2. CreateRoomResult 创建房间结果协议，由服务器受到CreateRoom协议后发往客户端，表示此次创建房间的行动是否成功。（谁发送CreateRoom协议，那么谁就会接受CreateRoomResult协议），参数如下：
	1. userName : string : 创建房间的用户名
	2. roomName : string : 创建的房间的名字
	3. RoomResult : string : "success"表示加入成功,"fail"表示加入失败
	4. FailReason : string : 失败原因

2. AttendRoom 加入房间协议,用户点击加入房间按钮,向服务器发送加入房间的消息,参数如下:
	1. roomName : string : 要加入的房间的名字
	2. userName : string : 要加入该房间的用户

3. AttendRoomResult 加入房间结果协议,用于返回客户端加入房间的信息,表示是否成功.同时,会返回给用户这个房间的具体情况,比如,具体有哪几个玩家,分别是什么阵营的,叫什么名字等等.需要注意的是,该消息会发送给目标房间内的所有玩家(用于更新他们的客户端)
	1. roomName : string : 要加入的房间的名字
	2. userName : string : 要加入该房间的用户 
	3. RoomResult : string : "success"表示加入成功,"fail"表示加入失败
	4. FailReason : string : 失败原因
	 
4. GetRoomList 获得所有房间的协议信息,用于返回目前服务器上所有的房间信息,协议参数如下:
	1. roomCount : int : 目前服务器一共有的房间数量,后面roomCount次数据都是下面这种格式
		1. roomName : string : 房间名
		2. roomPerson : int : 房间人数
		3. roomStatus : string : 房间游戏状态(是否已经开始游戏)
		
	
房间系统流程:

	玩家登录 -> 进入房间列表 -> 点击 加入房间 or 创建房间 -> 发送 CreateRoom 或 AttendRoom协议 -> 如果是CreateRoom协议，客户端要额外接受CreateRoomResult消息，
                                                                                           如果消息为Success，那么客户端再次发送AttendRoom协议，加入到自己创建的房间中。
	                |                                                   |
	                v                                                   v
               每隔N秒,发送GetRoomList协议,               等待AttendRoomResult协议信息,根据这个协议
               用于更新客户端房间列表                      将当前用户加入进目标房间(同时也将此用户的
                                                        channel连接加入房间系统的ChannelGroup中)
                                                                        |
                                                                        v
                                                        进入具体的房间,客户端进入房间内部,进行准备或
                                                        准备开始游戏


对上面协议的补充：

1. AttendRoomResult协议信息发送给所有在目标房间的玩家。当玩家受到一份AttendRoomResult协议时，首先观察userName是否和自己的userName相同，如果不相同，那么再观察roomResult，如果为Success，更新客户端，让房间再加入一个玩家。当userName与自己的userName相同并且roomResult为Success时，发送GetRoom协议，获得要加入的游戏房间的具体情况，更新游戏客户端。
2. 在房间列表视图中，每隔N秒发送一次GetRoomList协议，用于更新视图中的房间列表。


### 游戏逻辑层 协议设计 ###

#### 何时发送同步消息 ####
何时发布同步消息需要所有客户端和服务端约定好。

在这里我这样约定：

> 打开客户端并进行登录的玩家（称为玩家A），在客户端游玩过程中，被称为在本地客户端游玩，在这个客户端中生成的其他玩家的Clone单位，被称为远程单位。对于一个本地客户端来说，只有在 **玩家A发生了改变** 、 **玩家A对游戏世界造成了影响（即击中\治疗了某个单位）** 才发送消息，同步游戏逻辑。
> 
> 其他的诸如，远程玩家单位B和C在玩家A的本地客户端中相互攻击之类的事件，**不同步！！！** 
> 
> 因为按照这个约定，应该是玩家B在他的本地客户端对玩家C的Clone进行攻击，才会广播这个伤害事件给所有客户端。
> 
> 也就说，每个本地客户端只管自己的本地玩家对游戏世界造成的影响，而不考虑其他玩家所造成的影响，因为其他玩家造成的影响到时候他们会广播消息进来的！

#### 解耦 ####
首先分析，在整个游戏流程中，哪些游戏逻辑是需要同步的，哪些是不需要的，下面列出需要进行同步的游戏逻辑。

1. 位置同步√
2. 伤害同步√
3. 动画同步√
4. 状态同步（指 中毒状态、燃烧等战斗状态）**
5. 技能释放同步（ 当某个单位准备释放技能时，同步该技能，让其他玩家的客户端也能观察到玩家A对玩家B释放技能的现象 ）**
6. 胜负判断同步 ×
7. 物品同步（当某玩家获得某件物品时，此消息要广播到其他客户端，告诉他们玩家Z获得了xxx为物品）
8. 同步等级，当玩家A在客户端A升级后，应将此升级事件广播给客户端BCDEF....Z。√
9. 同步NPC的行为，对于非玩家单位，也需要进行同步，NPC需要有一个参照同步对象，在这里，我设置所有NPC的行为都同步至房主的NPC的活动。简而言之，当房主的客户端中NPC进行活动时（移动、攻击），将这些现象同步至所有成员的客户端中。减少误差。

不需要同步的：

1. 经济（即金币），对于每个玩家的金币，不需要同步，金币的作用是用来购买装备和消耗品，而物品我们已经进行同步，所以这里不需要在浪费资源为经济进行同步。
2. 死亡事件不需要同步，当玩家受到一定伤害导致HP归0时，玩家操纵的单位死亡，此时不需要同步死亡事件，因为伤害是同步的，所有客户度中，玩家A都受到了导致他HP归0的伤害，所以在所有客户端中，他都会死亡，并且等待系统将他复活，同时，这个复活的时间也是确定的，所以完全不需要同步死亡事件。
3. 单位属性（包括生命值魔法值）不需要同步，单位身上的属性由单位的等级和装备决定，这两者都在上面给予了同步。
4. 战争迷雾和小地图不需要同步，战争迷雾和小地图的显示由各个单位的位置决定，单位的位置在上面已经给出同步，所以战争迷雾和小地图也不需要同步。
5. UI不同步。。。额，这个应该是废话，UI肯定不同步。

那么，在本游戏中，最重要的一个问题就出现了，如何解耦 **网络通信的逻辑** 和 **正常游戏（单机）** 的逻辑


#### 同步位置 ####
1. UpdatePos协议，当玩家进行移动时，将会在每一帧向服务器发送UpdatePos协议，用于报告服务器自己当前位置。其参数如下：
	1. userName : string : 游戏用户的ID，唯一，用于标识正在移动的那个玩家
	2. roomName : string : 玩家所处房间的ID，用于保证服务器稍后回复的消息只在房间内玩家传递
	2. x : float : 单位的x坐标
	3. y : float : 单位的y坐标
	4. z : float : 单位的z坐标
	5. tx : float : 单位绕X轴旋转的角度
	6. ty : float : 单位绕y轴旋转的角度
	7. tz : float : 单位绕z轴旋转的角度

**移动式同步**

因为有网络延迟的存在，所以位置同步理论上不可能做到跟本地计算一样流畅，在客户端中，玩家可能会观察到其他玩家的移动带有卡顿，这是因为位置同步时，新的位置与旧位置距离过大产生。

在这里我采用的解决方法是，采用渐进移动的方式解决卡顿，简单来说，当要客户端受到位置同步的消息时，首先观察自身的位置和新位置的距离（不要开根号【根号消耗性能】，平方比较就好），如果距离的平方小于某个阈值，那么根据新旧位置的方向来渐进移动（通常很快速），如果距离的平方大于某个阈值（如玩家使用闪现技能），那么就直接将新位置赋给单位。

#### 伤害同步 ####
1. Damage协议，当某一个远程生成的玩家X在玩家A的本地客户端中被玩家A击中，生成一个 “玩家X被玩家A击中” 的伤害协议，并广播至所有客户端。协议参数如下：
	1. roomName : string : 攻击者所处的房间ID，限定了此协议只在房间内传递 
	1. Attacker : string : 施加伤害的单位的ID（即UserName）
	2. Victim ： string  : 受害人的ID（即UserName）
	3. BaseDamage : float : 本次造成的Damage中的Base伤害
	4. PlusDamage : float : 本次造成的Damage中的Plus伤害

当客户端收到Damage协议后，首先判断本地玩家是不是Attacker，如果是，不做处理（原因是，攻击事件，攻击者本地客户端已经处理，他需要的仅仅是要让其他客户端知道，有个人给我打了这样）。如果本地玩家不是Attacker，那么处理此次Damage消息，首先找到受害人，然后调用受害人的characterMono.Damage(new Damage(BaseDamge,PlusDamge));方法来实现伤害的同步。	

> 伤害同步协议的触发是单位收到伤害时触发，那么谁应该注册这个监听方法呢。
> 
> 答案是，所有单位都应该注册这个监听方法，在被攻击时判断攻击者是否是本地玩家，如果是本地玩家，那么发送Damage协议，然后本地客户端计算伤害。

#### 动画同步 ####
首先要明确的是，只有人物的动画是需要同步的，粒子特效动画不进行同步。

在本游戏中，所有人物的动画都由Animator组件进行启动，那么如果要同步，实际上就是发送一个消息，让其他客户端的本地玩家Clone也执行同样的操作。

1. AnimationOperation 动画协议,当本地玩家X开始播放一段动画的时候，发送该协议到服务端，让服务端广播至所有客户端，告诉他们“玩家X正在播放一段动画”。该协议参数如下：
	1. roomName : string : 播放动画的玩家X所在房间，限定此协议只在房间内传递
	2. userName : string : 标识玩家X
	3. operation : string : 动画操作，如 “run” 表示 aniamtor.setBool("run",true),"attack"表示animator.setTrigger("attack").

#### 同步等级 ####
1. Level 同步等级协议，参数如下：
	1. roomName : string : 房间名
	2. userName : string : 要同步等级的玩家的ID
	3. level : int : 要同步的等级

#### 物品同步 ####
1. AddItem 物品同步协议，参数如下：
	1. roomName : string : 房间名
	1. userName : string : 身上物品改变的用户
	2. itemId : int : 要获得的物品的ID,根据Id获得物品输入列表中的元物品,然后给这个英雄
	3. itemNumber : int : 物品数量
2. DeleteItem 物品删除协议,参数如下:
	1.  roomName : string : 房间名
	2.  userName : string : 身上物品改变的用户
	3.  itemId : int : 要删除的物品的ID


#### 技能同步 ####
2. SpellSkill 协议 技能释放同步协议,技能释放同步通过监听单位的Spell方法来实现,当本地玩家在Spell方法内准备释放某一个技能的时候,发送技能释放同步协议,协议参数如下:
	1. roomName : string : 房间名
	2. userName : string : 放技能的玩家的用户名  
	3. skillID : int : 释放的技能在技能列表中的编号 
	4. skillLevel : int : 释放的技能的等级(因为在技能列表的技能永远是0级状态)
	4. skillTarget : string : 表示技能的目标是什么,如果为"Target",那么表示技能目标是敌人,下面参数给出敌人ID,如果为"Position",那么表示技能目标是一个目标地点,下面给出三个float型参数表示目标位置
	5. targetID : string : 被释放技能的单位的用户名
	6. x : float : 表示目标位置的x坐标
	7. y : float : 表示目标位置的y坐标
	8. z : float : 表示目标位置的z坐标
	
#### NPC同步 ####
在网络游戏中,要保证每个NPC的行为在每个客户端都是一致的,这就需要每个NPC有一个参照物,也就是说,由一个客户端用于真正处理AI系统,其余所有客户端的NPC的行为都模仿自该客户端,一般而言,我将这个主客户端设为房主.

对于每一个NPC单位,它会以id为"NPC#xxx"加入networkPlayers字典中,他们的行为受到SynchronizeNPC类的监控,当触发 移动事件/伤害事件/释放技能事件 时,都会向服务器发送协议,以此来告诉其他模仿的客户端,NPC xxx 移动了/攻击了谁/放了什么技能.
