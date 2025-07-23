# ET框架Fiber间数据共享与通信

## 核心设计原则：Share Nothing, Message Everything

ET框架采用了**无共享架构**，Fiber之间完全隔离，通过**Actor消息传递**进行通信。这种设计避免了多线程编程的复杂性，提供了高并发、高可用的架构。

## 为什么不共享数据？

### 传统共享数据的问题
```csharp
// ❌ 传统多线程共享数据方式的问题
public class SharedDataProblems
{
    private static Dictionary<long, PlayerData> sharedPlayerData = new();
    private static readonly object lockObj = new object();
    
    public PlayerData GetPlayerData(long playerId)
    {
        lock (lockObj) // 锁竞争
        {
            return sharedPlayerData[playerId]; // 可能死锁
        }
    }
    
    // 问题：
    // 1. 锁竞争导致性能下降
    // 2. 死锁风险
    // 3. 难以调试和维护
    // 4. 扩展困难
}
```

### ET框架的解决方案
```csharp
// ✅ ET框架的消息传递方式
public class MessageBasedDataAccess
{
    public async ETTask<PlayerData> GetPlayerData(long playerId)
    {
        // 向数据管理Fiber发送请求
        ActorId dataManagerActor = GetDataManagerActor();
        var request = new GetPlayerDataRequest { PlayerId = playerId };
        var response = await MessageHelper.CallActor(dataManagerActor, request);
        return response.PlayerData;
    }
    
    // 优势：
    // 1. 无锁，天然线程安全
    // 2. 故障隔离
    // 3. 易于扩展和调试
    // 4. 可以分布到不同进程/机器
}
```

## 1. Actor消息传递模式（主要推荐）

### Actor化Entity
```csharp
// 让Entity成为Actor，可以接收消息
[EntitySystemOf(typeof(PlayerDataComponent))]
public static partial class PlayerDataComponentSystem
{
    [EntitySystem]
    private static void Awake(this PlayerDataComponent self)
    {
        // 挂载MailBoxComponent使其成为Actor
        self.AddComponent<MailBoxComponent, MailBoxType>(MailBoxType.OrderedMessage);
    }
}

public class PlayerDataComponent : Entity, IAwake
{
    public Dictionary<long, PlayerData> playerCache = new();
}
```

### 消息定义
```csharp
// 请求消息
[MemoryPackable]
public partial class GetPlayerDataRequest : MessageObject, IRequest
{
    public int RpcId { get; set; }
    public long PlayerId { get; set; }
}

// 响应消息
[MemoryPackable] 
public partial class GetPlayerDataResponse : MessageObject, IResponse
{
    public int RpcId { get; set; }
    public int Error { get; set; }
    public PlayerData PlayerData { get; set; }
}

// 通知消息
[MemoryPackable]
public partial class PlayerDataUpdateNotify : MessageObject, IMessage
{
    public long PlayerId { get; set; }
    public PlayerData UpdatedData { get; set; }
}
```

### 消息处理器
```csharp
[MessageHandler(SceneType.Map)]
public class GetPlayerDataRequestHandler : MessageHandler<Scene, GetPlayerDataRequest, GetPlayerDataResponse>
{
    protected override async ETTask Run(Scene scene, GetPlayerDataRequest request, GetPlayerDataResponse response)
    {
        // 从数据管理器获取数据
        PlayerDataComponent dataManager = scene.GetComponent<PlayerDataComponent>();
        
        PlayerData data;
        if (dataManager.playerCache.TryGetValue(request.PlayerId, out data))
        {
            response.PlayerData = data;
        }
        else
        {
            // 从数据库加载
            data = await LoadPlayerDataFromDB(request.PlayerId);
            dataManager.playerCache[request.PlayerId] = data;
            response.PlayerData = data;
        }
    }
}
```

### 发送消息
```csharp
public class MessageSendingExample
{
    public async ETTask RequestPlayerData(long playerId)
    {
        // 获取目标Actor的ID
        ActorId dataManagerActor = new ActorId(process, fiberId, entityInstanceId);

        // 发送请求消息并等待响应
        var request = new GetPlayerDataRequest { PlayerId = playerId };
        var response = await MessageHelper.CallActor(dataManagerActor, request) as GetPlayerDataResponse;
        
        if (response.Error == ErrorCore.ERR_Success)
        {
            PlayerData playerData = response.PlayerData;
            // 处理数据...
        }
    }
    
    public async ETTask NotifyPlayerDataUpdate(long playerId, PlayerData newData)
    {
        // 发送单向通知消息
        var notify = new PlayerDataUpdateNotify 
        { 
            PlayerId = playerId, 
            UpdatedData = newData 
        };
        
        // 通知所有相关的Fiber
        await MessageHelper.SendActor(uiFiberActor, notify);
        await MessageHelper.SendActor(battleFiberActor, notify);
    }
}
```

## 2. 数据Owner模式

### 明确的数据归属
```csharp
// 玩家数据的Owner Fiber
[EntitySystemOf(typeof(PlayerDataManagerComponent))]
public static partial class PlayerDataManagerComponentSystem
{
    [EntitySystem]
    private static void Awake(this PlayerDataManagerComponent self)
    {
        self.AddComponent<MailBoxComponent, MailBoxType>(MailBoxType.OrderedMessage);
    }
}

public class PlayerDataManagerComponent : Entity, IAwake
{
    // 这个Fiber是所有玩家数据的Owner
    private Dictionary<long, PlayerData> playerDataCache = new();
    private Dictionary<long, DateTime> lastUpdateTime = new();
    
    public async ETTask<PlayerData> GetPlayerData(long playerId)
    {
        if (!playerDataCache.TryGetValue(playerId, out PlayerData data))
        {
            // 从数据库加载
            data = await LoadFromDatabase(playerId);
            playerDataCache[playerId] = data;
        }
        return data;
    }
    
    public async ETTask UpdatePlayerData(long playerId, PlayerData newData)
    {
        playerDataCache[playerId] = newData;
        lastUpdateTime[playerId] = DateTime.Now;
        
        // 异步保存到数据库
        _ = SaveToDatabase(playerId, newData);
        
        // 通知相关Fiber数据已更新
        await NotifyDataUpdate(playerId, newData);
    }
}
```

### 数据消费者
```csharp
// 其他Fiber作为数据消费者，只能通过消息请求数据
public class GameLogicComponent : Entity
{
    public async ETTask ProcessPlayer(long playerId)
    {
        // 向数据Owner Fiber请求数据
        ActorId dataOwnerActor = GetPlayerDataManagerActor();
        var request = new GetPlayerDataRequest { PlayerId = playerId };
        var response = await MessageHelper.CallActor(dataOwnerActor, request);
        
        PlayerData data = response.PlayerData;
        
        // 处理业务逻辑
        ProcessPlayerLogic(data);
        
        // 如果需要修改数据，发送更新请求
        var updateRequest = new UpdatePlayerDataRequest 
        { 
            PlayerId = playerId, 
            NewData = modifiedData 
        };
        await MessageHelper.CallActor(dataOwnerActor, updateRequest);
    }
}
```

## 3. 事件通知模式

### 数据变更广播
```csharp
public class EventNotificationPattern
{
    public async ETTask UpdatePlayerExp(long playerId, int expGain)
    {
        // 1. 更新数据
        PlayerData playerData = await GetPlayerData(playerId);
        int oldExp = playerData.Exp;
        playerData.Exp += expGain;
        
        // 2. 保存数据
        await SavePlayerData(playerData);
        
        // 3. 创建事件通知
        var expChangeEvent = new PlayerExpChangedEvent
        {
            PlayerId = playerId,
            OldExp = oldExp,
            NewExp = playerData.Exp,
            ExpGain = expGain
        };
        
        // 4. 广播给所有相关Fiber
        var notificationTasks = new[]
        {
            MessageHelper.SendActor(uiFiberActor, expChangeEvent),        // 更新UI显示
            MessageHelper.SendActor(battleFiberActor, expChangeEvent),    // 更新战力
            MessageHelper.SendActor(achievementFiberActor, expChangeEvent), // 检查成就
            MessageHelper.SendActor(rankFiberActor, expChangeEvent)       // 更新排行榜
        };
        
        await ETTask.WaitAll(notificationTasks);
    }
}
```

### 事件处理器
```csharp
// UI Fiber处理经验变化事件
[MessageHandler(SceneType.ClientScene)]
public class PlayerExpChangedEventHandler : MessageHandler<Scene, PlayerExpChangedEvent>
{
    protected override async ETTask Run(Scene scene, PlayerExpChangedEvent message)
    {
        // 更新UI显示
        UIPlayerInfo uiPlayerInfo = scene.GetComponent<UIPlayerInfo>();
        uiPlayerInfo.UpdateExpDisplay(message.NewExp);
        
        // 播放经验获得特效
        await uiPlayerInfo.PlayExpGainEffect(message.ExpGain);
    }
}

// 战斗Fiber处理经验变化事件
[MessageHandler(SceneType.Map)]
public class BattleExpChangedEventHandler : MessageHandler<Scene, PlayerExpChangedEvent>
{
    protected override async ETTask Run(Scene scene, PlayerExpChangedEvent message)
    {
        // 重新计算战斗力
        BattleComponent battleComp = scene.GetComponent<BattleComponent>();
        await battleComp.RecalculatePower(message.PlayerId);
        
        // 检查是否升级
        if (IsLevelUp(message.OldExp, message.NewExp))
        {
            await ProcessLevelUp(message.PlayerId);
        }
    }
}
```

## 4. 缓存同步模式

### 本地缓存 + 消息同步
```csharp
public class CachedDataManager : Entity
{
    // 本地缓存，只读
    private Dictionary<long, PlayerBattleInfo> battleInfoCache = new();
    private Dictionary<long, DateTime> cacheTime = new();
    private readonly TimeSpan CACHE_EXPIRE_TIME = TimeSpan.FromMinutes(5);
    
    public async ETTask<PlayerBattleInfo> GetBattleInfo(long playerId)
    {
        // 检查缓存是否有效
        if (battleInfoCache.TryGetValue(playerId, out var cachedInfo) &&
            cacheTime.TryGetValue(playerId, out var time) &&
            DateTime.Now - time < CACHE_EXPIRE_TIME)
        {
            return cachedInfo; // 返回缓存数据
        }
        
        // 缓存过期或不存在，请求最新数据
        ActorId dataOwnerActor = GetPlayerDataOwnerActor();
        var request = new GetPlayerBattleInfoRequest { PlayerId = playerId };
        var response = await MessageHelper.CallActor(dataOwnerActor, request);
        
        // 更新缓存
        battleInfoCache[playerId] = response.BattleInfo;
        cacheTime[playerId] = DateTime.Now;
        
        return response.BattleInfo;
    }
    
    // 接收数据更新通知，使缓存失效
    [MessageHandler]
    public void HandlePlayerInfoUpdate(PlayerInfoUpdateNotify notify)
    {
        // 移除缓存，强制下次重新获取
        battleInfoCache.Remove(notify.PlayerId);
        cacheTime.Remove(notify.PlayerId);
        
        Log.Debug($"Player {notify.PlayerId} cache invalidated due to data update");
    }
}
```

## 5. 数据流水线模式

### 阶段性数据处理
```csharp
public class DataPipelineExample
{
    // 数据处理流水线：收集 -> 分析 -> 计算 -> 存储
    public async ETTask ProcessPlayerAction(PlayerAction action)
    {
        // 阶段1：数据收集Fiber
        ActorId collectorActor = GetDataCollectorActor();
        var collectMsg = new CollectActionDataMessage 
        { 
            Action = action,
            Timestamp = DateTime.Now
        };
        await MessageHelper.SendActor(collectorActor, collectMsg);
        
        // 后续阶段由事件驱动自动触发
    }
}

// 数据收集Fiber
[MessageHandler(SceneType.DataCollector)]
public class CollectActionDataHandler : MessageHandler<Scene, CollectActionDataMessage>
{
    protected override async ETTask Run(Scene scene, CollectActionDataMessage message)
    {
        // 收集和预处理数据
        ActionData processedData = PreprocessActionData(message.Action);
        
        // 发送到分析阶段
        ActorId analyzerActor = GetDataAnalyzerActor();
        var analyzeMsg = new AnalyzeDataMessage { Data = processedData };
        await MessageHelper.SendActor(analyzerActor, analyzeMsg);
    }
}

// 数据分析Fiber
[MessageHandler(SceneType.DataAnalyzer)]
public class AnalyzeDataHandler : MessageHandler<Scene, AnalyzeDataMessage>
{
    protected override async ETTask Run(Scene scene, AnalyzeDataMessage message)
    {
        // 分析数据，提取有用信息
        AnalysisResult result = AnalyzeData(message.Data);
        
        // 发送到计算阶段
        ActorId calculatorActor = GetDataCalculatorActor();
        var calculateMsg = new CalculateDataMessage { AnalysisResult = result };
        await MessageHelper.SendActor(calculatorActor, calculateMsg);
    }
}
```

## 6. 读写分离模式

### 专门的读写Fiber
```csharp
// 写操作Fiber - 负责所有数据修改
public class DataWriteManager : Entity
{
    private Queue<WriteOperation> writeQueue = new();
    
    [MessageHandler]
    public async ETTask<UpdatePlayerDataResponse> HandleUpdatePlayerData(UpdatePlayerDataRequest request)
    {
        try
        {
            // 执行写操作
            await ExecuteWriteOperation(request);
            
            // 通知读Fiber缓存失效
            ActorId readManagerActor = GetDataReadManagerActor();
            var invalidateMsg = new InvalidateCacheMessage { PlayerId = request.PlayerId };
            await MessageHelper.SendActor(readManagerActor, invalidateMsg);
            
            // 广播数据变更事件
            await BroadcastDataChangeEvent(request.PlayerId, request.NewData);
            
            return new UpdatePlayerDataResponse { Success = true };
        }
        catch (Exception ex)
        {
            Log.Error($"Update player data failed: {ex}");
            return new UpdatePlayerDataResponse { Success = false, Error = ex.Message };
        }
    }
    
    private async ETTask ExecuteWriteOperation(UpdatePlayerDataRequest request)
    {
        // 写入数据库
        await database.UpdatePlayerDataAsync(request.PlayerId, request.NewData);
        
        // 写入缓存
        await UpdateCache(request.PlayerId, request.NewData);
    }
}

// 读操作Fiber - 负责所有数据查询
public class DataReadManager : Entity
{
    private LRUCache<long, PlayerData> readCache = new(capacity: 10000);
    
    [MessageHandler]
    public async ETTask<GetPlayerDataResponse> HandleGetPlayerData(GetPlayerDataRequest request)
    {
        // 先查本地缓存
        if (readCache.TryGet(request.PlayerId, out PlayerData cachedData))
        {
            return new GetPlayerDataResponse { PlayerData = cachedData };
        }
        
        // 缓存未命中，从数据库读取
        PlayerData data = await database.LoadPlayerDataAsync(request.PlayerId);
        
        // 更新缓存
        readCache.Add(request.PlayerId, data);
        
        return new GetPlayerDataResponse { PlayerData = data };
    }
    
    [MessageHandler]
    public void HandleInvalidateCache(InvalidateCacheMessage message)
    {
        // 使缓存失效
        readCache.Remove(message.PlayerId);
        Log.Debug($"Cache invalidated for player {message.PlayerId}");
    }
}
```

## 7. 实际应用场景

### MMO游戏玩家数据架构
```csharp
public class MMOPlayerDataArchitecture
{
    public async ETTask SetupPlayerDataSystem()
    {
        // 创建专门的数据管理Fiber
        await CreateDataManagementFibers();
        
        // 设置数据同步机制
        await SetupDataSynchronization();
    }
    
    private async ETTask CreateDataManagementFibers()
    {
        // 玩家基础数据Fiber
        await FiberManager.Instance.Create(
            SchedulerType.Thread, 1, SceneType.PlayerData, "PlayerDataManager");
        
        // 背包数据Fiber
        await FiberManager.Instance.Create(
            SchedulerType.Thread, 1, SceneType.Inventory, "InventoryManager");
        
        // 公会数据Fiber
        await FiberManager.Instance.Create(
            SchedulerType.Thread, 1, SceneType.Guild, "GuildManager");
        
        // 好友数据Fiber
        await FiberManager.Instance.Create(
            SchedulerType.Thread, 1, SceneType.Friend, "FriendManager");
    }
    
    // UI需要显示完整玩家信息时的并行数据获取
    public async ETTask<CompletePlayerInfo> LoadCompletePlayerInfo(long playerId)
    {
        // 并行请求多个Fiber的数据
        var tasks = new ETTask[]
        {
            RequestPlayerBaseInfo(playerId),
            RequestPlayerInventory(playerId),
            RequestPlayerGuildInfo(playerId),
            RequestPlayerFriendList(playerId)
        };
        
        await ETTask.WaitAll(tasks);
        
        // 组合所有数据
        return new CompletePlayerInfo
        {
            BaseInfo = (tasks[0] as ETTask<PlayerBaseInfo>).Result,
            Inventory = (tasks[1] as ETTask<PlayerInventory>).Result,
            GuildInfo = (tasks[2] as ETTask<PlayerGuildInfo>).Result,
            FriendList = (tasks[3] as ETTask<PlayerFriendList>).Result
        };
    }
}
```

### 实时战斗数据同步
```csharp
public class RealTimeBattleDataSync
{
    public async ETTask ProcessDamage(long attackerId, long targetId, int damage)
    {
        // 战斗Fiber计算伤害
        BattleResult result = CalculateDamage(attackerId, targetId, damage);
        
        // 创建伤害事件
        var damageEvent = new DamageEvent
        {
            AttackerId = attackerId,
            TargetId = targetId,
            Damage = result.FinalDamage,
            IsCritical = result.IsCritical,
            Timestamp = DateTime.Now
        };
        
        // 并行通知所有相关Fiber
        var notificationTasks = new[]
        {
            // 通知表现Fiber播放特效
            MessageHelper.SendActor(viewFiberActor, new PlayDamageEffectMessage(damageEvent)),
            
            // 通知数据Fiber更新血量
            MessageHelper.SendActor(dataFiberActor, new UpdateHealthMessage(targetId, -result.FinalDamage)),
            
            // 通知AI Fiber更新仇恨值
            MessageHelper.SendActor(aiFiberActor, new UpdateHatredMessage(attackerId, targetId, result.FinalDamage)),
            
            // 通知统计Fiber记录数据
            MessageHelper.SendActor(statsFiberActor, new RecordDamageMessage(damageEvent))
        };
        
        await ETTask.WaitAll(notificationTasks);
    }
}
```

## 总结和最佳实践

### 优势
1. **线程安全**: 无共享数据，天然线程安全
2. **故障隔离**: 一个Fiber崩溃不影响其他Fiber的数据
3. **扩展性**: 可以轻松拆分到不同进程/机器
4. **调试友好**: 消息传递易于追踪和调试
5. **性能优秀**: 避免锁竞争，消息传递开销很小

### 关键原则
- **永远不要直接访问其他Fiber的对象**
- **所有跨Fiber操作都通过消息完成**
- **数据有明确的Owner，其他Fiber只能请求**
- **合理设计消息粒度，避免过于频繁的通信**
- **善用缓存减少跨Fiber通信次数**

### 性能优化建议
1. **批量操作**: 将多个小操作合并为一个消息
2. **异步通知**: 不需要返回值的操作使用Send而不是Call
3. **缓存策略**: 在消费者端维护合适的缓存
4. **消息池化**: 重用消息对象减少GC压力

这种设计让ET框架实现了真正的高并发、高可用架构，是现代分布式系统设计的优秀实践。