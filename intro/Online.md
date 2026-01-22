---
layout: page
title: Online
parent: intro
nav_order: 34
---

* TOC
{:toc}

# ProviderWithOnline 和 Online 使用说明文档

## 文档目标
分析 `ProviderWithOnline.java` 和 `Online.java` 的设计意图、用法、核心接口，并输出使用说明文档。

## 分析结果

### 一、ProviderWithOnline.java - 在线服务提供者基类

#### 1.1 设计意图
`ProviderWithOnline` 是一个继承自 `ProviderImplement` 的服务提供者基类，专门用于管理在线玩家状态的服务。它的核心设计目标是：

- **Online实例管理**：管理一个默认的Online实例和多个命名的OnlineSet（在线集合）
- **负载监控**：集成ProviderLoadWithOnline实现负载监控和报告
- **断线处理**：统一处理连接断开事件，维护玩家在线状态

#### 1.2 使用场景

**场景1：MMORPG游戏服务器**
- **需求**：一个大型MMORPG需要支持数万玩家同时在线，需要实时管理玩家状态，处理登录、登出、断线重连
- **为什么使用**：提供了完整的在线状态管理框架，自动处理玩家会话，不需要开发者自己维护复杂的连接状态
- **实现方式**：继承ProviderWithOnline，创建一个默认Online管理游戏主流程

**场景2：多客户端/多平台同时在线管理**
- **需求**：游戏支持多个客户端同时在线（PC端、移动端、网页端），或者同一账号使用不同的应用（游戏App、独立聊天App、社区App），每个客户端需要独立的在线状态管理
- **为什么使用**：支持创建多个OnlineSet，每个OnlineSet独立管理不同客户端的在线状态，实现跨客户端通信和状态同步
- **实现方式**：

```java
// 创建多个OnlineSet对应不同的客户端平台
provider.create(app, "PC", "Mobile", "Web", "ChatApp");

// 示例：玩家同时在PC端和移动端登录
// - PC OnlineSet：管理PC端的在线状态、配置、数据
// - Mobile OnlineSet：管理移动端的在线状态、配置、数据
// - 两个OnlineSet独立管理，互不干扰
// - 可以互相通信（如PC端通知移动端上线）
// - 可以数据同步（如从PC端同步进度到移动端）

// 示例：游戏App和独立聊天App同时在线
// - GameApp OnlineSet：管理游戏内的在线状态
// - ChatApp OnlineSet：管理独立聊天App的在线状态
// - 玩家可以在游戏内，聊天App离线
// - 或者聊天App在线，游戏App离线（独立聊天客户端）
// - 两者可以互相通信（游戏内聊天同步到聊天App）
```

**场景3：服务器负载均衡**
- **需求**：需要根据在线人数动态调整负载，实现跨服务器的负载调度
- **为什么使用**：集成了ProviderLoadWithOnline，自动报告和监控每个服务器的在线人数和负载
- **实现方式**：通过getLoad()获取负载信息，配合负载均衡策略使用

**场景4：分布式游戏世界**
- **需求**：游戏世界分布在多个服务器上，需要统一管理玩家的全局在线状态
- **为什么使用**：OnlineShared数据在全局共享，可以追踪玩家在任何服务器的状态
- **实现方式**：通过OnlineShared查询玩家全局状态，实现跨服功能

#### 1.3 核心成员

```java
protected Online online;  // 默认的Online实例
private ProviderLoadWithOnline load;  // 负载监控
protected final HashMap<String, Online> onlineSetMap;  // 所有Online集合
```

#### 1.3 核心接口

**获取Online实例**
- `getOnline()`: 获取默认Online实例
- `getOnline(String name)`: 获取指定名称的Online实例
- `foreachOnline(Consumer<Online> consumer)`: 遍历所有Online实例

**生命周期管理**
- `create(AppBase app, String... names)`: 创建默认Online和指定名称的OnlineSet
- `start()`: 启动Online和负载监控
- `stop()`: 停止所有Online服务

**事件处理**
- `ProcessLinkBroken(LinkBroken p)`: 处理连接断开事件

#### 1.4 使用方式

```java
// 1. 继承ProviderWithOnline
public class GameProvider extends ProviderWithOnline {
    // 自定义实现
}

// 2. 在App.Start流程中创建
provider.create(app, "OnlineSetName1", "OnlineSetName2");

// 3. 启动服务
provider.start();

// 4. 获取Online实例
Online defaultOnline = provider.getOnline();
Online namedOnline = provider.getOnline("OnlineSetName1");
```

---

### 二、Online.java - 在线状态管理核心类

#### 2.1 设计意图
`Online` 是整个在线状态管理的核心类，负责：

- **玩家在线状态管理**：登录、登出、断线重连、超时处理
- **本地数据管理**：管理角色在当前服务器进程的本地数据（Local）
- **协议发送**：提供单发、群发、广播、可靠通知等多种协议发送方式
- **多OnlineSet支持**：支持创建多个独立的在线集合，实现业务隔离
- **热更新支持**：支持热更新场景下的数据迁移和恢复
- **负载均衡转发**：支持跨服务器的请求转发（Transmit）

#### 2.2 使用场景

**场景1：玩家登录登出管理**
- **需求**：玩家登录游戏时需要初始化数据，登出时需要保存数据并清理资源
- **为什么使用**：提供完整的登录登出流程管理，自动处理会话状态、版本控制、断线重连
- **实现方式**：

```java
// 注册登录事件
online.getLoginEvents().add((sender, arg) -> {
    // 初始化玩家数据
    initPlayerData(arg.roleId);
    // 加载玩家背包
    loadInventory(arg.roleId);
    // 通知好友玩家上线
    notifyFriendsOnline(arg.roleId);
    return 0;
});

// 注册登出事件
online.getLogoutEvents().add((sender, arg) -> {
    // 保存玩家数据
    savePlayerData(arg.roleId);
    // 清理缓存
    clearCache(arg.roleId);
    // 通知好友玩家下线
    notifyFriendsOffline(arg.roleId);
    return 0;
});
```

**场景2：断线重连与延迟登出**
- **需求**：玩家网络波动导致短暂断线，需要自动恢复会话，避免数据丢失和游戏体验中断
- **为什么使用**：提供延迟登出机制，断线后等待一段时间才真正登出，期间重连可恢复会话
- **实现方式**：

```java
// 配置延迟登出时间（如30秒）
// 在Zeze配置中：OnlineLogoutDelay=30000

// 注册断线事件
online.getLinkBrokenEvents().add((sender, arg) -> {
    // 保存关键数据（保险起见）
    saveCriticalData(arg.roleId);
    // 通知客户端连接断开（显示重连按钮）
    notifyClientLinkBroken(arg.roleId);
    // 不清理数据，等待可能的重连
    return 0;
});

// 注册重连事件（ReLogin）
online.getReloginEvents().add((sender, arg) -> {
    // 恢复会话
    restoreSession(arg.roleId);
    // 同步断线期间的数据变更
    syncDataDuringDisconnect(arg.roleId);
    return 0;
});
```

**场景3：向玩家发送协议**
- **需求**：需要向玩家发送各种游戏通知、奖励、更新等
- **为什么使用**：提供多种发送方式（单发、群发、广播），支持事务中发送，支持可靠通知
- **实现方式**：

```java
// 场景3.1：发送奖励通知（普通发送）
public void giveReward(long roleId, Reward reward) {
    // 发放奖励
    addRewardToInventory(roleId, reward);

    // 事务提交后发送通知
    Transaction.whileCommit(() -> {
        online.send(roleId, new RewardNotify(reward));
    });
}

// 场景3.2：发送重要通知（可靠通知，必须送达）
public void sendImportantAnnouncement(long roleId, String announcement) {
    // 添加可靠通知标记
    online.addReliableNotifyMark(roleId, "SystemAnnouncement");

    // 发送可靠通知
    online.sendReliableNotify(roleId, "SystemAnnouncement",
        new AnnouncementProtocol(announcement));
}

// 场景3.3：广播服务器公告（所有在线玩家）
public void broadcastServerNotice(String notice) {
    online.broadcast(new ServerNotice(notice));
}

// 场景3.4：群发给公会成员（多个玩家）
public void sendToGuildMembers(long guildId, GuildMessage msg) {
    List<Long> members = getGuildMembers(guildId);
    online.send(members, new GuildMessageNotify(msg));
}
```

**场景4：本地数据管理**
- **需求**：某些数据只需要在单个服务器进程中使用，不需要全局共享，用于提高性能
- **为什么使用**：提供Local数据存储，仅本进程可见，减少网络开销和数据库压力
- **实现方式**：

```java
// 场景4.1：缓存战斗数据（仅战斗服务器需要）
public void startBattle(long roleId, BattleInstance battle) {
    // 保存战斗数据到本地（快，不跨进程）
    online.setLocalBean(roleId, "CurrentBattle", battle);

    // 战斗过程中频繁访问，不需要全局同步
    // ...
}

public BattleInstance getBattle(long roleId) {
    // 从本地获取（快速）
    return online.getLocalBean(roleId, "CurrentBattle", BattleInstance.class);
}

// 场景4.2：临时会话数据（聊天、交易等）
public void startTrade(long roleId1, long roleId2, TradeSession session) {
    // 保存交易会话数据
    online.setLocalBean(roleId1, "TradeSession", session);
    online.setLocalBean(roleId2, "TradeSession", session);
}

// 场景4.3：玩家登出时清理
online.getLocalRemoveEvents().add((sender, arg) -> {
    // 清理本地数据
    var battle = online.getLocalBean(arg.roleId, "CurrentBattle", BattleInstance.class);
    if (battle != null) {
        endBattle(arg.roleId, battle);
    }
    return 0;
});
```

**场景5：跨服务器请求转发**
- **需求**：需要查询或操作在另一个服务器上的玩家数据
- **为什么使用**：提供Transmit机制，自动找到目标玩家所在服务器并转发请求
- **实现方式**：

```java
// 场景5.1：跨服好友查询
public class FriendService {
    public void init() {
        // 注册跨服查询处理器
        online.getTransmitActions().put("QueryFriendProfile", (sender, target, param) -> {
            // 查询目标玩家的资料
            var profile = getProfile(target);

            // 返回给查询发起者
            online.send(sender, new FriendProfileResponse(target, profile));
            return 0;
        });
    }

    // 查询好友资料（好友可能在其他服务器）
    public void queryFriendProfile(long requester, long friendRoleId) {
        // Transmit会自动找到friendRoleId所在的服务器并转发请求
        online.transmit(requester, "QueryFriendProfile", friendRoleId);
    }
}

// 场景5.2：跨服邮件发送
online.getTransmitActions().put("SendMail", (sender, target, param) -> {
    var mail = decodeMail(param);
    deliverMail(target, mail);
    return 0;
});

// 发送邮件给离线玩家（无论他在哪台服务器）
public void sendMail(long sender, long targetRoleId, Mail mail) {
    online.transmit(sender, "SendMail", targetRoleId, encodeMail(mail));
}
```

**场景6：多客户端/多平台同时在线**
- **需求**：玩家可能同时在多个客户端登录（PC端、移动端、网页端），或者在同一账号下使用不同的应用（游戏App、聊天App、社区App），需要独立的在线状态管理
- **为什么使用**：支持创建多个OnlineSet，每个OnlineSet独立管理不同客户端的在线状态，互不干扰
- **实现方式**：

```java
// 创建多个OnlineSet对应不同的客户端
provider.create(app, "PC", "Mobile", "Web", "ChatApp");

// 场景6.1：PC端和移动端同时在线
public class MultiPlatformGame {
    private Online pcOnline;
    private Online mobileOnline;

    public void init() {
        pcOnline = provider.getOnline("PC");
        mobileOnline = provider.getOnline("Mobile");

        // PC端登录事件
        pcOnline.getLoginEvents().add((sender, arg) -> {
            // 加载PC端配置
            loadPCSettings(arg.roleId);
            // 同步数据到PC端（从移动端同步过来）
            syncDataFromMobile(arg.roleId);
            // 通知移动端：PC端上线了
            notifyCrossPlatform(arg.roleId, "PC", "Online");
            return 0;
        });

        // 移动端登录事件
        mobileOnline.getLoginEvents().add((sender, arg) -> {
            // 加载移动端配置
            loadMobileSettings(arg.roleId);
            // 同步数据到移动端（从PC端同步过来）
            syncDataFromPC(arg.roleId);
            // 通知PC端：移动端上线了
            notifyCrossPlatform(arg.roleId, "Mobile", "Online");
            return 0;
        });

        // PC端登出事件
        pcOnline.getLogoutEvents().add((sender, arg) -> {
            // 保存PC端数据到服务器
            savePCData(arg.roleId);
            // 通知移动端：PC端下线了
            notifyCrossPlatform(arg.roleId, "PC", "Offline");
            return 0;
        });
    }

    // 跨平台数据同步
    private void syncDataFromMobile(long roleId) {
        var mobileOnline = provider.getOnline("Mobile");
        if (mobileOnline.isLogin(roleId)) {
            // 从移动端获取最新数据
            var mobileData = mobileOnline.getUserData(roleId, GameData.class);
            // 同步到PC端
            pcOnline.setUserData(roleId, mobileData);
        }
    }
}

// 场景6.2：游戏App和独立聊天App同时在线
public class GameAndChatApps {
    private Online gameOnline;
    private Online chatOnline;

    public void init() {
        gameOnline = provider.getOnline("GameApp");
        chatOnline = provider.getOnline("ChatApp");

        // 游戏App登录
        gameOnline.getLoginEvents().add((sender, arg) -> {
            // 初始化游戏数据
            initGameData(arg.roleId);
            // 如果聊天App也在线，通知游戏内好友状态
            if (chatOnline.isLogin(arg.roleId)) {
                syncChatStatusToGame(arg.roleId);
            }
            return 0;
        });

        // 聊天App登录（独立App）
        chatOnline.getLoginEvents().add((sender, arg) -> {
            // 加载聊天记录和好友列表
            loadChatData(arg.roleId);
            // 如果游戏App也在线，显示游戏内状态
            if (gameOnline.isLogin(arg.roleId)) {
                showInGameStatus(arg.roleId);
            }
            return 0;
        });

        // 跨App消息通知
        chatOnline.getLinkBrokenEvents().add((sender, arg) -> {
            // 聊天App断线，在游戏内显示通知
            if (gameOnline.isLogin(arg.roleId)) {
                gameOnline.send(arg.roleId, new ChatAppDisconnectedNotice());
            }
            return 0;
        });
    }

    // 从游戏App发送聊天消息到聊天App
    public void sendGameChatMessage(long roleId, String message) {
        var chatOnline = provider.getOnline("ChatApp");
        if (chatOnline.isLogin(roleId)) {
            // 在聊天App中显示游戏内聊天
            chatOnline.send(roleId, new GameChatMessage(message));
        }
    }
}

// 场景6.3：网页端临时查看，不影响PC端/移动端
public class WebViewer {
    private Online webOnline;
    private Online pcOnline;
    private Online mobileOnline;

    public void handleWebLogin(long roleId) {
        webOnline = provider.getOnline("Web");

        // 网页端登录（只读模式）
        webOnline.getLoginEvents().add((sender, arg) -> {
            // 仅加载只读数据
            loadReadOnlyData(arg.roleId);

            // 查询主设备状态
            String primaryDevice = null;
            if (pcOnline.isLogin(arg.roleId)) {
                primaryDevice = "PC";
            } else if (mobileOnline.isLogin(arg.roleId)) {
                primaryDevice = "Mobile";
            }

            // 通知网页端：这是临时查看会话
            webOnline.send(arg.roleId, new WebViewerNotice(primaryDevice));
            return 0;
        });

        // 网页端登出（不影响主设备）
        webOnline.getLogoutEvents().add((sender, arg) -> {
            // 清理临时数据即可
            clearTempData(arg.roleId);
            return 0;
        });
    }
}

// 场景6.4：跨设备踢人逻辑
public class MultiDeviceKick {
    private Online pcOnline;
    private Online mobileOnline;

    // PC端登录时，如果移动端已登录，询问是否踢掉移动端
    public void onPCLogin(long roleId) {
        if (mobileOnline.isLogin(roleId)) {
            // 通知移动端：PC端登录了
            mobileOnline.send(roleId, new AnotherDeviceLogin("PC"));

            // 可选：自动踢掉移动端
            // mobileOnline.kick(roleId, BKick.ErrorDuplicateLogin, "PC端登录");
        }
    }

    // 强制登出所有设备
    public void kickAllDevices(long roleId, String reason) {
        pcOnline.kick(roleId, BKick.ErrorCustom, reason);
        mobileOnline.kick(roleId, BKick.ErrorCustom, reason);
    }
}
```

**场景7：热更新数据迁移**
- **需求**：游戏运行时需要热更新Bean类，需要迁移现有数据到新的Bean结构
- **为什么使用**：提供热更新支持，自动迁移Local数据中的Bean
- **实现方式**：

```java
// 注册自定义Bean类型
Online.register(PlayerDataV1.class);
Online.register(PlayerDataV2.class);

// 热更新时执行数据迁移
online.upgrade(oldBean -> {
    if (oldBean instanceof PlayerDataV1) {
        var v1 = (PlayerDataV1)oldBean;
        // 迁移到V2
        var v2 = new PlayerDataV2();
        v2.setLevel(v1.getLevel());
        v2.setExp(v1.getExp());
        // V2新增字段
        v2.setNewField(defaultValue);
        return v2;
    }
    return null; // 不迁移
});
```

**场景8：负载均衡与服务器选择**
- **需求**：根据服务器在线人数和负载，动态分配玩家到不同服务器
- **为什么使用**：提供负载查询接口，配合ProviderLoad实现负载均衡
- **实现方式**：

```java
// 选择负载最低的服务器
public int selectBestServer() {
    int bestServerId = -1;
    int minLoad = Integer.MAX_VALUE;

    for (var server : providerApp.providerDirectService.providerByServerId.values()) {
        var load = provider.getLoad().getOnlineLoad(server.getServerId());
        if (load < minLoad) {
            minLoad = load;
            bestServerId = server.getServerId();
        }
    }

    return bestServerId;
}

// 玩家登录时分配到负载低的服务器
public int assignServerForLogin(long roleId) {
    int serverId = selectBestServer();
    // 通知玩家连接到指定服务器
    return serverId;
}
```

#### 2.3 核心数据结构

Online系统使用5个核心数据表来管理玩家的在线状态和数据，每个表都有特定的设计意图和使用场景。

---

##### 2.3.1 _tOnline（角色在线信息表）

**设计意图**
存储角色在当前服务器进程的在线信息，每个服务器进程有独立的副本，不与其他服务器共享。

**为什么需要**
- **ServerId追踪**：记录角色当前登录在哪个服务器进程上
- **可靠通知支持**：管理可靠通知的标记和索引
- **本地数据隔离**：每个服务器进程可以存储不同的本地数据
- **性能优化**：不需要跨进程同步，访问速度快

**存储内容**

```java
public class BOnline extends Bean {
    private int serverId;                    // 当前所在服务器ID
    private HashSet<String> reliableNotifyMark; // 可靠通知标记集合
    private long reliableNotifyIndex;        // 可靠通知索引
    private long reliableNotifyConfirmIndex; // 已确认的可靠通知索引
    private Bean userData;                   // 用户自定义数据（全局可见）
}
```

**使用场景**

**场景1：查询角色所在服务器**

```java
// 查询角色在哪个服务器上
public int locateRoleServer(long roleId) {
    var online = online.getOnline(roleId);
    if (online != null) {
        return online.getServerId();
    }
    return -1; // 不在线
}

// 用于跨服功能：如跨服组队时找到角色所在服务器
```

**场景2：可靠通知管理**

```java
// 添加可靠通知标记（确保重要通知送达）
online.addReliableNotifyMark(roleId, "ImportantAnnouncement");
online.sendReliableNotify(roleId, "ImportantAnnouncement",
    new MaintenanceNotice(server, time));

// 客户端确认后，自动删除已送达的通知
```

**场景3：跨服务器请求定位**

```java
// 发送跨服请求前，先定位目标角色
public void sendCrossServerRequest(long requester, long target) {
    var targetOnline = online.getOnline(target);
    if (targetOnline == null) {
        // 目标不在线
        online.send(requester, new TargetOfflineResponse());
        return;
    }

    if (targetOnline.getServerId() != getCurrentServerId()) {
        // 目标在其他服务器，使用Transmit转发
        online.transmit(requester, "QueryInfo", target);
    } else {
        // 目标在本地服务器，直接处理
        processLocalRequest(requester, target);
    }
}
```

**数据库表名**
- 表名：`Zeze.Game.Online.tOnline_[OnlineSetName]`
- 默认OnlineSet：`Zeze.Game.Online.tOnline_`
- 命名OnlineSet：`Zeze.Game.Online.tOnline_Mobile`
- 特点：每个服务器进程有独立的本地缓存，不与其他服务器共享

---

##### 2.3.2 _tOnlineShared（角色共享在线信息表）

**设计意图**
存储角色的全局共享在线信息，所有服务器进程都能访问，用于维护角色的全局在线状态和会话信息。

**为什么需要**
- **全局在线状态**：所有服务器都能查询角色的在线状态
- **会话管理**：管理角色的登录链接信息（LinkName、LinkSid）
- **版本控制**：通过LoginVersion/LogoutVersion防止并发冲突
- **账号绑定**：维护角色与账号的绑定关系
- **跨服协作**：多个服务器需要协同管理同一个角色

**存储内容**

```java
public class BOnlineShared extends Bean {
    private String account;           // 绑定的账号
    private BLink link;               // 登录链接信息（LinkName、LinkSid、State）
    private long loginVersion;        // 登录版本号（每次登录递增）
    private long logoutVersion;       // 登出版本号
    private Bean userData;            // 用户自定义数据（全局可见）
}

public class BLink extends Bean {
    private String linkName;          // 连接名称（如"Link-GameServer1"）
    private long linkSid;             // 连接会话ID
    private int state;                // 状态：eOffline/eLogined/eLinkBroken
}
```

**使用场景**

**场景1：全局在线状态查询**

```java
// 查询角色是否在线（全局）
public boolean isOnline(long roleId) {
    var onlineShared = online.getOnlineShared(roleId);
    return onlineShared != null
        && onlineShared.getLink().getState() == eLogined;
}

// 用于好友列表、公会成员列表等显示在线状态
```

**场景2：防止重复登录**

```java
// Login流程中检查是否已登录
public long ProcessLoginRequest(Login rpc) {
    var onlineShared = online.getOrAddOnlineShared(roleId);
    var link = onlineShared.getLink();

    // 如果已登录，踢掉旧连接
    if (link.getState() == eLogined) {
        providerApp.providerService.kick(
            link.getLinkName(),
            link.getLinkSid(),
            BKick.ErrorDuplicateLogin,
            "duplicate role login"
        );
    }

    // 创建新登录
    // ...
}
```

**场景3：版本控制防止并发冲突**

```java
// Local数据带版本号，过期自动删除
public void processLoginVersion(long roleId) {
    var onlineShared = online.getOnlineShared(roleId);
    var local = online._tlocal.get(roleId);

    // 版本不一致，说明是过期数据，删除
    if (local != null
        && local.getLoginVersion() != onlineShared.getLoginVersion()) {
        online._tlocal.remove(roleId);
    }
}
```

**场景4：断线重连恢复**

```java
// ReLogin时检查版本号，确保恢复正确的会话
public long ProcessReLoginRequest(ReLogin rpc) {
    var onlineShared = online.getOrAddOnlineShared(roleId);

    // 比对版本号，防止重连到错误的会话
    long expectedVersion = rpc.Argument.getLoginVersion();
    if (onlineShared.getLoginVersion() != expectedVersion) {
        // 版本不匹配，拒绝重连
        return errorCode(ResultCodeVersionMismatch);
    }

    // 恢复会话...
}
```

**数据库表名**
- 表名：`Zeze.Game.Online.tOnlineShared_[OnlineSetName]`
- 默认OnlineSet：`Zeze.Game.Online.tOnlineShared_`
- 命名OnlineSet：`Zeze.Game.Online.tOnlineShared_Mobile`
- 特点：全局共享，所有服务器进程都能访问，通过数据库同步

---

##### 2.3.3 _tlocal（角色本地数据表）

**设计意图**
存储角色在当前服务器进程的本地数据，仅本进程可见，不与其他服务器共享，用于缓存和性能优化。

**为什么需要**
- **性能优化**：频繁访问的数据不需要跨进程通信
- **存储临时数据**：战斗数据、临时会话等不需要全局共享的数据
- **减少网络开销**：本地数据读写速度快，不占用网络带宽
- **版本控制**：带LoginVersion，过期自动清理
- **超时清理**：长时间不活跃的数据自动删除

**存储内容**

```java
public class BLocal extends Bean {
    private long loginVersion;        // 登录版本号（与OnlineShared一致）
    private BLink link;               // 本地链接信息副本
    private HashMap<String, BAny> datas; // 本地数据集合（Key-Value）
}

public class BAny extends Bean {
    private Any any;                  // 存储任意Bean类型
}
```

**使用场景**

**场景1：战斗数据缓存**

```java
// 开始战斗，保存战斗数据到本地（快）
public void startBattle(long roleId, BattleInstance battle) {
    online.setLocalBean(roleId, "CurrentBattle", battle);

    // 战斗过程中频繁访问，不需要跨进程通信
}

// 战斗结束，清理数据
public void endBattle(long roleId) {
    online.removeLocalBean(roleId, "CurrentBattle");
}
```

**场景2：临时会话数据**

```java
// 开始交易，保存交易会话
public void startTrade(long roleId1, long roleId2, TradeSession session) {
    online.setLocalBean(roleId1, "TradeSession", session);
    online.setLocalBean(roleId2, "TradeSession", session);
}

// 玩家登出时自动清理（超时或版本过期）
online.getLocalRemoveEvents().add((sender, arg) -> {
    var trade = online.getLocalBean(arg.roleId, "TradeSession", TradeSession.class);
    if (trade != null) {
        cancelTrade(arg.roleId, trade);
    }
    return 0;
});
```

**场景3：模块加载缓存**

```java
// 热更新模块缓存
public class GameModule {
    public void loadModuleData(long roleId) {
        // 加载模块数据到本地缓存
        var data = new ModuleData();
        // ... 加载数据

        online.setLocalBean(roleId, "ModuleData", data);
    }

    public ModuleData getModuleData(long roleId) {
        // 优先从本地缓存获取
        return online.getLocalBean(roleId, "ModuleData", ModuleData.class);
    }
}
```

**场景4：超时自动清理**

```java
// 配置超时时间（如10分钟不活跃就清理）
online.setLocalActiveTimeout(600 * 1000);
online.setLocalCheckPeriod(600 * 1000);

// 活跃时更新时间
online.setLocalActiveTimeIfPresent(roleId);

// 超时后自动删除，释放内存
```

**数据库表名**
- 表名：`Zeze.Game.Online.tlocal_[OnlineSetName]`
- 默认OnlineSet：`Zeze.Game.Online.tlocal_`
- 命名OnlineSet：`Zeze.Game.Online.tlocal_Mobile`
- 特点：仅本进程可见，不与其他服务器共享，带版本控制和超时清理

---

##### 2.3.4 _tRoleTimers（角色定时器表）

**设计意图**
管理角色的在线定时器，角色在线时触发的定时任务存储在这个表中，角色登出后自动清理。

**为什么需要**
- **在线期间定时任务**：如心跳检查、定时保存、定时刷新等
- **自动清理**：角色登出后，定时器自动删除，不会占用资源
- **角色级别隔离**：每个角色的定时器独立管理，互不干扰
- **支持暂停/恢复**：角色登出时暂停，重新登录时恢复

**使用场景**

**场景1：定时保存数据**

```java
// 每5分钟自动保存角色数据
public void startAutoSave(long roleId) {
    online.getTimerRole().schedule(roleId, 5 * 60 * 1000, (context) -> {
        // 保存角色数据
        saveRoleData(roleId);
        return 0; // 返回0表示继续定时
    });
}
```

**场景2：定时刷新状态**

```java
// 每1秒刷新战斗状态
public void startBattleTick(long roleId) {
    online.getTimerRole().schedule(roleId, 1000, (context) -> {
        var battle = online.getLocalBean(roleId, "Battle", BattleInstance.class);
        if (battle != null) {
            battle.tick(); // 战斗逻辑帧
        }
        return 0;
    });
}
```

**场景3：定时检查**

```java
// 每30秒检查玩家是否在安全区
public void startSafetyZoneCheck(long roleId) {
    online.getTimerRole().schedule(roleId, 30 * 1000, (context) -> {
        if (isInSafetyZone(roleId)) {
            // 在安全区，恢复血量
            recoverHealth(roleId);
        }
        return 0;
    });
}
```

**数据库表名**
- 表名：`Zeze.Game.Online.tRoleTimers_[OnlineSetName]`
- 默认OnlineSet：`Zeze.Game.Online.tRoleTimers_`
- 命名OnlineSet：`Zeze.Game.Online.tRoleTimers_Mobile`
- 特点：角色登出后自动清理，仅存储在线期间的定时器

---

##### 2.3.5 _tRoleOfflineTimers（角色离线定时器表）

**设计意图**
管理角色的离线定时器，角色登出后仍需要执行的定时任务存储在这个表中，即使角色离线也能触发。

**为什么需要**
- **离线期间定时任务**：如离线经验增长、建筑升级、资源产出等
- **持久化存储**：角色登出后定时器仍然存在，服务器重启后继续执行
- **跨服务器生效**：无论角色在哪个服务器登录，离线定时器都有效
- **一次性触发**：触发后自动删除，不会重复执行

**使用场景**

**场景1：离线经验增长**

```java
// 玩家离线后，每10分钟获得离线经验
public void startOfflineExp(long roleId) {
    var timer = new OfflineExpTimer();
    timer.setRoleId(roleId);
    timer.setExpPerMinute(100);

    // 登出10分钟后开始计算离线经验
    online._tRoleOfflineTimers.getOrAdd(roleId).setTimer(timer);
}

// 触发离线经验
public void onOfflineExpTimer(long roleId) {
    var timer = online._tRoleOfflineTimers.get(roleId);
    if (timer != null) {
        long offlineMinutes = calculateOfflineTime(roleId);
        long exp = offlineMinutes * timer.getExpPerMinute();

        // 玩家重新登录时发放离线经验
        addOfflineExp(roleId, exp);

        // 删除定时器（一次性）
        online._tRoleOfflineTimers.remove(roleId);
    }
}
```

**场景2：建筑升级**

```java
// 玩家开始建造建筑，需要1小时
public void startBuilding(long roleId, int buildingId) {
    var timer = new BuildingTimer();
    timer.setRoleId(roleId);
    timer.setBuildingId(buildingId);
    timer.setCompleteTime(System.currentTimeMillis() + 60 * 60 * 1000);

    online._tRoleOfflineTimers.getOrAdd(roleId).setTimer(timer);
}

// 建筑完成（玩家可能在线，也可能离线）
public void onBuildingComplete(long roleId) {
    var timer = online._tRoleOfflineTimers.get(roleId);
    if (timer != null && timer instanceof BuildingTimer) {
        var building = (BuildingTimer)timer;

        // 建筑完成，发放奖励
        completeBuilding(roleId, building.getBuildingId());

        // 删除定时器
        online._tRoleOfflineTimers.remove(roleId);
    }
}
```

**场景3：邮件/通知延迟发送**

```java
// 1小时后发送提醒邮件
public void scheduleReminderMail(long roleId, String content) {
    var timer = new ReminderMailTimer();
    timer.setRoleId(roleId);
    timer.setContent(content);
    timer.setSendTime(System.currentTimeMillis() + 60 * 60 * 1000);

    online._tRoleOfflineTimers.getOrAdd(roleId).setTimer(timer);
}
```

**数据库表名**
- 表名：`Zeze.Game.Online.tRoleOfflineTimers_[OnlineSetName]`
- 默认OnlineSet：`Zeze.Game.Online.tRoleOfflineTimers_`
- 命名OnlineSet：`Zeze.Game.Online.tRoleOfflineTimers_Mobile`
- 特点：持久化存储，角色离线后仍然存在，触发后自动删除

---

#### 2.4 数据表对比总结

| 表名 | 作用域 | 生命周期 | 用途 | 数据库共享 |
|------|--------|----------|------|-----------|
| **_tOnline** | 本进程 | 角色在当前进程在线时 | ServerId、可靠通知标记、本地UserData | 否（各进程独立） |
| **_tOnlineShared** | 全局 | 角色在线期间 | 全局在线状态、链接信息、版本号、账号 | 是（全局共享） |
| **_tlocal** | 本进程 | 角色在当前进程在线时 | 临时缓存、战斗数据、会话数据 | 否（各进程独立） |
| **_tRoleTimers** | 全局 | 角色在线期间 | 在线定时任务（心跳、刷新、保存） | 是（跟随角色） |
| **_tRoleOfflineTimers** | 全局 | 离线定时任务触发前 | 离线定时任务（离线经验、建筑升级） | 是（跟随角色） |

**状态枚举**

```java
eOffline = 0;     // 离线
eLogined = 1;     // 已登录
eLinkBroken = 2;  // 断线（延迟登出中）
```

#### 2.3 核心接口分类

##### 2.3.1 Online状态查询

```java
// 查询Online状态
Online getOnline(long roleId);
boolean isOnline(long roleId);
boolean isLogin(long roleId);

// 获取共享状态
BOnlineShared getOnlineShared(long roleId);
BOnlineShared getLoginOnlineShared(long roleId);  // 仅登录状态

// 获取状态值
int getState(long roleId);
Long getLoginVersion(long roleId);
Long getLogoutVersion(long roleId);
```

##### 2.3.2 用户数据管理

```java
// UserData - 跟随OnlineShared，所有服务器可见
<T extends Bean> T getUserData(long roleId);
void setUserData(long roleId, T data);

// LocalBean - 仅本进程可见
<T extends Bean> T getLocalBean(long roleId, String key);
void setLocalBean(long roleId, String key, T bean);
void removeLocalBean(long roleId, String key);
<T extends Bean> T getOrAddLocalBean(long roleId, String key, T defaultHint);
```

##### 2.3.3 协议发送接口

**单发协议**

```java
// 发送给单个角色
void send(long roleId, Protocol<?> p);
void sendOnline(long roleId, Protocol<?> p);

// 发送RPC
void sendRpc(long roleId, Rpc<A, R> rpc, ProtocolHandle<Rpc<A, R>> handle);
TaskCompletionSource<R> sendOnlineRpcForWait(long roleId, Rpc<A, R> rpc);
```

**群发协议**

```java
// 发送给多个角色
void send(Collection<Long> roleIds, Protocol<?> p);
void sendOnline(Collection<Long> roleIds, Protocol<?> p);

// 发送给所有OnlineSet
void sendAllOnlines(long roleId, Protocol<?> p);
void sendAllOnlines(Collection<Long> roleIds, Protocol<?> p);
```

**可靠通知**（保证送达，需要确认）

```java
void addReliableNotifyMark(long roleId, String listenerName);
void removeReliableNotifyMark(long roleId, String listenerName);
void sendReliableNotify(long roleId, String listenerName, Protocol<?> p);
void sendReliableNotifyWhileCommit(long roleId, String listenerName, Protocol<?> p);
```

**广播协议**（发送给所有连接的Link）

```java
int broadcast(Protocol<?> p);
int broadcast(Protocol<?> p, int timeout);
int broadcast(Protocol<?> p, boolean onlySameVersion);
```

**直接发送**（通过LinkName+LinkSid发送，不经过Online查询）

```java
boolean send(String linkName, long linkSid, Protocol<?> p);
boolean send(Long roleId, String linkName, long linkSid, Protocol<?> p);
```

**事务相关发送**

```java
void sendWhileCommit(long roleId, Protocol<?> p);           // 事务提交时发送
void sendWhileRollback(long roleId, Protocol<?> p);         // 事务回滚时发送
```

##### 2.3.4 事件系统

```java
// 登录事件
EventDispatcher getLoginEvents();

// 断线重连事件
EventDispatcher getReloginEvents();

// 登出事件
EventDispatcher getLogoutEvents();

// 本地数据删除事件
EventDispatcher getLocalRemoveEvents();

// 断线事件
EventDispatcher getLinkBrokenEvents();
```

##### 2.3.5 请求转发（Transmit）

```java
// 注册转发处理器
ConcurrentHashMap<String, TransmitAction> getTransmitActions();

// 转发请求
void transmit(long sender, String actionName, long roleId);
void transmit(long sender, String actionName, long roleId, Serializable parameter);
void transmit(long sender, String actionName, Iterable<Long> roleIds);

// 事务相关转发
void transmitWhileCommit(long sender, String actionName, long roleId);
void transmitWhileRollback(long sender, String actionName, long roleId);
```

TransmitAction接口：

```java
public interface TransmitAction {
    long call(long sender, long target, Binary parameter) throws Exception;
}
```

##### 2.3.6 踢人功能

```java
void kick(long roleId, int code, String desc);
```

##### 2.3.7 OnlineSet管理

```java
String getOnlineSetName();
Online getOnline(String name);
Online getOnlineByContext();  // 获取上下文中的Online
```

#### 2.4 登录登出流程

**Login流程**
1. 创建Local记录（如不存在）
2. 验证账号绑定
3. 如果已登录，触发Logout并重试
4. 更新LoginVersion
5. 设置Link状态为eLogined
6. 设置Link的UserState（OnlineSetName + Context=roleId）
7. 清空可靠通知标记和队列
8. 触发Login事件

**ReLogin流程**（断线重连）
1. 类似Login流程
2. 保留ServerId
3. 同步可靠通知队列（ReliableNotifySync）

**Logout流程**
1. 设置LogoutVersion
2. 设置Link状态为eOffline
3. 删除Local记录
4. 触发Logout事件
5. 清空UserState

**LinkBroken流程**（断线）
1. 设置Link状态为eLinkBroken
2. 触发LinkBroken事件
3. 延迟登出（配置OnlineLogoutDelay时间后执行Logout）
4. 如果在延迟期间重连成功，取消Logout

#### 2.5 本地数据超时清理

Online会定期检查Local数据的活跃度，超时未活跃的数据会被清理：

```java
// 配置超时时间和检查间隔
online.setLocalActiveTimeout(600 * 1000);  // 活跃超时时间（默认10分钟）
online.setLocalCheckPeriod(600 * 1000);    // 检查间隔（默认10分钟）

// 更新活跃时间
online.setLocalActiveTimeIfPresent(roleId);
```

#### 2.6 多OnlineSet机制

Online支持创建多个独立的在线集合，实现业务隔离：

```java
// 创建多个OnlineSet
provider.create(app, "Game", "Chat", "Match");

// 每个OnlineSet独立管理角色在线状态
Online gameOnline = provider.getOnline("Game");
Online chatOnline = provider.getOnline("Chat");

// 角色可以同时在多个OnlineSet中登录
// 例如：角色在Game中登录，但不在Chat中登录
```

使用场景：
- 不同业务模块隔离（游戏逻辑、聊天、匹配等）
- 不同服务类型隔离（实时战斗服务、挂机服务等）
- 不同玩家群体隔离

#### 2.7 热更新支持

Online支持热更新场景下的Bean类型迁移：

```java
// 注册自定义Bean类型
Online.register(MyCustomBean.class);

// 热更新时的数据迁移
online.upgrade(oldBean -> {
    // 返回新的Bean实例
    return newBean;
});
```

### 三、何时应该使用这套系统

#### 3.1 判断标准

**应该使用 ProviderWithOnline + Online 的情况：**

1. **需要管理玩家在线状态**
   - 玩家需要登录、登出
   - 需要追踪玩家是否在线
   - 需要处理断线重连
   - 示例：MMORPG、卡牌对战、MOBA等所有需要持久连接的游戏

2. **需要向玩家推送数据**
   - 服务器主动发送通知给玩家
   - 需要实时更新游戏状态
   - 需要广播服务器公告
   - 示例：奖励通知、系统公告、好友上线通知

3. **需要跨服务器交互**
   - 玩家分布在多个游戏服务器
   - 需要跨服查询、跨服聊天、跨服交易
   - 需要全局玩家状态
   - 示例：大型MMO、分区游戏、跨服活动

4. **需要本地缓存优化性能**
   - 某些数据访问频繁，不需要全局同步
   - 需要减少网络开销和数据库压力
   - 示例：战斗数据、临时会话数据

5. **需要业务隔离**
   - 不同模块需要独立的在线状态
   - 需要独立的资源管理
   - 示例：独立聊天系统、独立匹配系统

**不应该使用的情况：**

1. **无状态服务**
   - HTTP REST API服务
   - 纯请求-响应模式
   - 不需要维护会话状态

2. **单机小游戏**
   - 所有逻辑在客户端
   - 服务器仅提供简单的数据存储

3. **实时性要求极高的场景**
   - FPS游戏（需要专用游戏服务器架构）
   - 需要微秒级延迟同步（Online系统的延迟可能过高）

#### 3.2 架构对比

**传统架构 vs Online系统架构**

| 特性 | 传统架构 | Online系统架构 |
|------|---------|--------------|
| 会话管理 | 需要自己维护 | 自动管理，带版本控制 |
| 断线重连 | 需要自己实现 | 延迟登出，自动恢复 |
| 跨服通信 | 需要自己实现 | Transmit自动转发 |
| 协议发送 | 需要自己实现连接管理 | 提供多种发送方式 |
| 负载监控 | 需要自己实现 | 集成ProviderLoad |
| 本地缓存 | 需要自己实现 | Local数据自动管理 |
| 可靠通知 | 需要自己实现 | 内置可靠通知机制 |
| 热更新 | 需要自己处理数据迁移 | 提供upgrade接口 |

**什么时候使用Online系统的价值最大：**

- 团队规模小，需要快速开发
- 需要支持复杂的在线状态管理
- 需要跨服务器功能
- 需要稳定的断线重连机制
- 需要负载均衡和动态扩展

### 四、典型使用场景

```java
public class GameApp {
    private ProviderWithOnline provider;

    public void start() {
        // 1. 创建Provider并配置OnlineSet
        provider.create(app, "Game", "Chat", "Match");

        // 2. 注册事件处理器
        var online = provider.getOnline("Game");
        online.getLoginEvents().add((sender, arg) -> {
            // 处理登录事件
            System.out.println("Role login: " + arg.roleId);
            return 0;
        });

        online.getLogoutEvents().add((sender, arg) -> {
            // 处理登出事件
            System.out.println("Role logout: " + arg.roleId);
            return 0;
        });

        // 3. 注册Transmit处理器
        online.getTransmitActions().put("QueryRoleInfo", (sender, target, param) -> {
            // 查询角色信息并返回给sender
            var roleInfo = getRoleInfo(target);
            online.send(sender, new RoleInfoResponse(roleInfo));
            return 0;
        });

        // 4. 启动服务
        provider.start();
    }
}
```

#### 3.2 发送协议给玩家

```java
public void sendReward(long roleId, Reward reward) {
    var online = provider.getOnline();

    // 在事务中发送
    Transaction.whileCommit(() -> {
        // 发放奖励后发送通知
        online.send(roleId, new RewardNotify(reward));
    });
}

// 可靠通知（必须送达）
public void sendImportantNotice(long roleId, String notice) {
    var online = provider.getOnline();

    // 添加标记
    online.addReliableNotifyMark(roleId, "ImportantNotice");

    // 发送可靠通知
    online.sendReliableNotify(roleId, "ImportantNotice",
        new NoticeProtocol(notice));
}

// 广播给所有在线玩家
public void broadcastAnnouncement(String announcement) {
    var online = provider.getOnline();
    online.broadcast(new AnnouncementProtocol(announcement));
}
```

#### 3.3 管理玩家本地数据

```java
public void savePlayerBattleData(long roleId, BattleData data) {
    var online = provider.getOnline();

    // 保存本地数据（仅本进程可见）
    online.setLocalBean(roleId, "BattleData", data);

    // 保存共享数据（所有服务器可见）
    online.setUserData(roleId, new UserData(data));
}

public BattleData getPlayerBattleData(long roleId) {
    var online = provider.getOnline();

    // 优先从本地获取
    var localData = online.getLocalBean(roleId, "BattleData", BattleData.class);
    if (localData != null) {
        return localData;
    }

    // 从共享数据获取
    return online.getUserData(roleId, BattleData.class);
}
```

#### 3.4 跨服务器请求转发

```java
public class QueryService {
    public void init() {
        var online = provider.getOnline();

        // 注册转发处理器
        online.getTransmitActions().put("QueryFriendList", (sender, target, param) -> {
            // 查询目标玩家的好友列表
            var friendList = getFriendList(target);

            // 返回给查询发起者
            online.send(sender, new FriendListResponse(target, friendList));
            return 0;
        });
    }

    public void queryFriendList(long requester, long target) {
        var online = provider.getOnline();

        // 转发请求到目标角色所在的服务器
        online.transmit(requester, "QueryFriendList", target);
    }
}
```

### 四、关键设计模式

#### 4.1 版本号机制
- `LoginVersion`: 每次登录递增，用于标识登录会话
- `LogoutVersion`: 登出时设置为当前LoginVersion
- Local数据带有LoginVersion，过期自动删除

#### 4.2 三层数据隔离
- `tOnline`: 每个服务器进程独立存储
- `tOnlineShared`: 全局共享（通过数据库同步）
- `tlocal`: 本地进程数据（用于缓存和优化）

#### 4.3 延迟登出机制
- 断线后不立即登出，等待配置的超时时间
- 超时期间重连成功可恢复会话
- 避免短暂网络抖动导致的数据丢失

#### 4.4 可靠通知机制
- 服务端维护通知队列
- 客户端按序确认接收
- 重连时自动同步未确认的通知

### 五、配置项

在Zeze配置中可设置：

```xml
<!-- Online登出延迟时间（毫秒） -->
<property name="OnlineLogoutDelay" value="30000" />

<!-- 本地数据活跃超时时间（毫秒） -->
<!-- 通过代码设置：online.setLocalActiveTimeout(600000) -->

<!-- 本地数据检查间隔（毫秒） -->
<!-- 通过代码设置：online.setLocalCheckPeriod(600000) -->
```

### 六、注意事项

1. **OnlineSet创建时机**：必须在App.Start流程中、Application.start()之前创建
2. **默认Online**：空字符串名称的Online为默认Online，负责管理所有OnlineSet
3. **线程安全**：所有接口都是线程安全的，可在多线程环境中调用
4. **事务使用**：建议在事务中使用Online接口，保证数据一致性
5. **Local数据限制**：Local数据仅本进程可见，不要存储需要跨进程访问的数据
6. **可靠通知限制**：使用可靠通知前需要先添加标记

---
