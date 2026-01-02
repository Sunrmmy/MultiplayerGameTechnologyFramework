# UE5 多人游戏基础

## 一、LAN局域网联机
![alt text](img/LAN联机代码.png)

主机通过以listen监听模式打开关卡，客户端通过主机IP地址连接主机。

## 二、Online Subsystem子系统
**基本结构和用途**
基本模块 OnlineSubsystem 定义服务指定的模块，并在引擎中进行注册。在初始化期间，在线子系统将尝试加载"Engine.ini"文件中指定的默认在线服务模块。对在线服务的所有访问都将通过此模块。

### 1. Steam Online Subsystem
我们可以通过Steam在线子系统托管游戏会话，主机在Steam上通过游戏会话开启监听服务器，其他玩家也能加入这个会话。
UE内置了支持多个平台的在线子系统，这些子系统能让我们针对不同平台配置项目，无需编写平台专属代码，无需直接对接平台API，只需配置项目启用对应平台的在线子系统即可。

具体步骤：
1. 添加Steam插件：
    - Online Subsystem Steam
    - Steam Socket（UE5.6+）
2. 在项目配置文件DefaultEngine.ini中添加网络驱动（NetDriver）和一些参数配置让项目适配Steam：
```ini
[/Script/Engine.GameEngine]
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemSteam.SteamNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")

[OnlineSubsystem]
DefaultPlatformService=Steam

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480
bInitServerOnClient=true

[/Script/OnlineSubsystemSteam.SteamNetDriver]
NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
```
> 可以在配置文件DefaultGame.ini中添加一些游戏会话参数配置，例如游戏最多联机人数等

3. 添加Mutiiplayer Sessions插件
4. 确保Steam客户端在后台运行：
    - 确保所有玩家Steam设置中的下载区域相同

## 三、Actor Replication（复制）

- Actor复制的相关属性：
    - **NetLoadOnClient（客户端网络加载）**：地图加载时是否在客户端加载，false则只在服务器存在
    - **Replication（复制）**：
        - 将Actor设为可复制对象，只要变量被标记为可复制，该变量数据就会在服务器和客户端之间同步。
        - 会强制启用客户端网络加载
    - **Replication Movement**：Actor启用复制并不会自动同步移动，需要单独启用移动复制

### 1. NetRole（网络角色）
- LocalRole（ENetRole）：表示当前机器上该 Actor 的角色。
- RemoteRole（ENetRole）：表示其他机器上该 Actor 的对应副本的角色。
![alt text](img/ENetRole.png)

1. 服务器上的玩家角色（权威角色）
LocalRole = ROLE_Authority
RemoteRole = ROLE_AutonomousProxy（对拥有该角色的客户端）或 ROLE_SimulatedProxy（对其他客户端）
✅ 服务器拥有最终决策权，负责同步状态给所有客户端。

2. 客户端上的本地玩家角色（自主代理）
LocalRole = ROLE_AutonomousProxy
RemoteRole = ROLE_Authority
✅ 客户端可以处理输入（如移动、开火），但关键操作需提交服务器验证。

3. 客户端上的其他玩家角色（模拟代理）
LocalRole = ROLE_SimulatedProxy
RemoteRole = ROLE_Authority
✅ 仅从服务器接收位置、动画等状态，进行插值平滑显示，不能控制。

4. 专用服务器上的非玩家 Actor（如掉落物）
LocalRole = ROLE_Authority
RemoteRole = ROLE_SimulatedProxy
✅ 服务器控制物理状态，客户端只做视觉模拟。

HasAuthority()函数用于检查当前Actor是否具有权限角色。

### 2. Attachment（附加）
- Actor Attachment：Actor附加也能被复制，但需要Actor本身是可复制的，附加才能复制
    AttachToActor函数
- Component Attachment：组件需要启用复制
    AttachToComponent函数

### 3. 变量复制

**第1步：重写 GetLifetimeReplicatedProps**
```cpp
virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
```
✅ 作用：
这是 UE 网络复制系统的“注册入口”。引擎在 Actor 初始化时会调用此函数，收集所有需要复制的属性列表。

🔍 原理：
每个 AActor 子类都可以通过重写此函数，告诉引擎：“我有哪些变量需要同步”。
引擎内部会缓存这个列表，在后续网络更新时只同步这些变量。
如果不重写此函数，即使变量标记为 Replicated，也不会被同步

📌 关键点：
必须调用 Super::GetLifetimeReplicatedProps(...)，以继承父类的复制属性（如 ACharacter 的位置、旋转等）。
此函数只在 服务器权威（Authority） 下生效。

**第2步：声明变量并标记 UPROPERTY(Replicated)**

**第3步：在 .cpp 中调用 DOREPLIFETIME 宏**
```cpp
// 在 .cpp 文件中（通常在 GetLifetimeReplicatedProps 实现内）
void AMutiplayer_CppCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    DOREPLIFETIME(this, Armor); // ← 这就是第3步
}
```
✅ 作用：
正式将 Armor 注册到网络复制系统中，使其成为“受管复制属性”

🔍 原理：
DOREPLIFETIME(Class, Property) 是一个宏，展开后会向 OutLifetimeProps 数组添加一条 FLifetimeProperty 记录。
引擎根据此记录监听 Armor 的值变化，当服务器上 Armor 改变时，自动打包新值发送给客户端，客户端收到后自动赋值（无需手动处理）

📌 关键细节：
必须写在 GetLifetimeReplicatedProps 函数体内
类名必须与当前类一致，可以使用this
此宏只处理简单类型（如 float, int32, bool, FVector 等）。复杂对象（如 UObject*）需用 DOREPLIFETIME_CONDITION 等高级宏。

### 4. RepNotify（复制通知）