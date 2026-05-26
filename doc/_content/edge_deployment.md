# 边缘部署改造计划（移除Redis依赖）

## 1. 概述

本改造计划旨在将WVP-PRO适配为边缘单机部署模式，移除Redis依赖，同时保留国标级联能力。

### 1.1 背景

- **边缘部署场景**：边缘设备数量有限，无需多WVP集群
- **级联需求**：边缘节点需向上级平台注册，实现目录、位置、报警推送
- **资源约束**：边缘环境资源有限，避免引入Redis等外部依赖

### 1.2 架构分析

```
┌─────────────────────────────────────────────────────────────┐
│                     WVP-PRO 架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  国标级联（SIP协议）              多WVP集群（RedisRpc）        │
│  ┌─────────────────┐              ┌─────────────────┐       │
│  │ 边缘节点 ──────→│ 上级平台     │ WVP-1 ←────Redis→ WVP-2  │
│  │ (不依赖Redis)   │              │ (依赖Redis)      │        │
│  └─────────────────┘              └─────────────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**关键发现：**

| 功能 | 通信方式 | Redis依赖 |
|------|---------|-----------|
| 边缘→上级级联 | SIP协议 | ❌ 不需要 |
| 平台注册/注销 | SIP协议 | ❌ 不需要 |
| 目录推送 | SIP NOTIFY | ❌ 不需要 |
| 位置推送 | SIP NOTIFY | ❌ 不需要 |
| 报警推送 | SIP NOTIFY | ❌ 不需要 |
| 设备点播/回放 | SIP INVITE | ❌ 不需要 |
| 集群内部分片 | RedisRpc | ✅ 需要 |

## 2. 改造范围

### 2.1 保留功能（边缘核心）

- ✅ 国标级联注册/注销
- ✅ 设备注册/心跳/离线
- ✅ 通道目录推送
- ✅ 位置信息推送
- ✅ 报警事件推送
- ✅ 设备点播/回放/下载
- ✅ 云台控制
- ✅ 语音广播/喊话
- ✅ 本地设备管理

### 2.2 移除功能（集群相关）

- ❌ RedisRpc跨节点通信
- ❌ 多WVP集群内部分片
- ❌ 跨服务器平台管理

### 2.3 替换组件

| 组件 | 原实现 | 替换方案 |
|------|--------|---------|
| 设备状态缓存 | Redis Hash | ConcurrentHashMap |
| 会话信息缓存 | Redis Hash | ConcurrentHashMap |
| 流信息缓存 | Redis Hash | ConcurrentHashMap |
| GPS位置缓存 | Redis String | ConcurrentHashMap |
| SIP CSEQ | Redis String | AtomicInteger |
| 消息订阅 | Redis Pub/Sub | Spring事件机制 |

## 3. 实施步骤

### 步骤1：创建本地缓存实现类

**文件：** `src/main/java/com/genersoft/iot/vmp/storager/impl/LocalCatchStorageImpl.java`

```java
package com.genersoft.iot.vmp.storager.impl;

import com.genersoft.iot.vmp.storager.IRedisCatchStorage;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * 本地缓存实现，用于边缘单机部署
 */
@Service
@ConditionalOnProperty(name = "wvp.cache.type", havingValue = "local", matchIfMissing = false)
public class LocalCatchStorageImpl implements IRedisCatchStorage {

    private final AtomicLong cseqCounter = new AtomicLong(1);
    private final ConcurrentHashMap<String, Device> deviceMap = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, InviteInfo> inviteMap = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, MediaInfo> streamMap = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, GPSMsgInfo> gpsMap = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, StreamAuthorityInfo> authorityMap = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, SendRtpInfo> sendRtpMap = new ConcurrentHashMap<>();

    @Override
    public Long getCSEQ() {
        return cseqCounter.getAndIncrement();
    }

    @Override
    public void updateDevice(Device device) {
        deviceMap.put(device.getDeviceId(), device);
    }

    @Override
    public Device getDevice(String deviceId) {
        return deviceMap.get(deviceId);
    }

    @Override
    public void removeDevice(String deviceId) {
        deviceMap.remove(deviceId);
    }

    // ... 实现其他IRedisCatchStorage接口方法
}
```

### 步骤2：禁用RedisRpc相关组件

**文件：** `src/main/java/com/genersoft/iot/vmp/conf/redis/RedisRpcConfig.java`

```java
@Component
@ConditionalOnProperty(name = "wvp.cache.type", havingValue = "redis", matchIfMissing = true)
public class RedisRpcConfig implements MessageListener {
    // 原有实现
}
```

### 步骤3：修改消息监听配置

**文件：** `src/main/java/com/genersoft/iot/vmp/conf/redis/RedisMsgListenConfig.java`

```java
@Configuration
@ConditionalOnProperty(name = "wvp.cache.type", havingValue = "redis", matchIfMissing = true)
@Order(value=1)
public class RedisMsgListenConfig {
    // 原有Redis消息监听配置
}
```

### 步骤4：添加Spring事件替代消息订阅

**文件：** `src/main/java/com/genersoft/iot/vmp/event/LocalEventPublisher.java`

```java
@Component
@ConditionalOnProperty(name = "wvp.cache.type", havingValue = "local", matchIfMissing = false)
public class LocalEventPublisher {

    private final ApplicationEventPublisher eventPublisher;

    public void sendAlarmMsg(AlarmChannelMessage msg) {
        eventPublisher.publishEvent(new AlarmEvent(msg));
    }

    public void sendStreamChangeMsg(String type, JSONObject jsonObject) {
        eventPublisher.publishEvent(new StreamChangeEvent(type, jsonObject));
    }
}
```

**文件：** `src/main/java/com/genersoft/iot/vmp/event/listener/LocalAlarmEventListener.java`

```java
@Component
@ConditionalOnProperty(name = "wvp.cache.type", havingValue = "local", matchIfMissing = false)
public class LocalAlarmEventListener {

    @EventListener
    public void onAlarmEvent(AlarmEvent event) {
        // 本地处理报警事件
    }
}
```

### 步骤5：配置文件修改

**文件：** `src/main/resources/application-local.yml`

```yaml
# 边缘单机部署配置
spring:
  redis:
    enabled: false

wvp:
  cache:
    type: local  # local或redis
  server:
    id: edge-001  # 边缘节点唯一标识

# 级联上级平台配置
sip:
  show-ip: true
  domain: 3402000000
  id: 34020000002000000001
  password: admin123
```

### 步骤6：修改依赖调用点

**文件：** `src/main/java/com/genersoft/iot/vmp/gb28181/service/impl/PlatformServiceImpl.java`

```java
@Service
public class PlatformServiceImpl implements IPlatformService {

    @Autowired(required = false)  // 本地模式可为null
    private IRedisRpcService redisRpcService;

    public boolean update(Platform platform) {
        if (redisRpcService != null && !userSetting.getServerId().equals(platform.getServerId())) {
            return redisRpcService.updatePlatform(platform.getServerId(), platform);
        }
        // 本地更新逻辑
        return platformMapper.update(platform);
    }
}
```

### 步骤7：移除RedisRpcController

移除以下文件或添加条件装配：
- `RedisRpcChannelPlayController.java`
- `RedisRpcDeviceController.java`
- `RedisRpcPlatformController.java`
- `RedisRpcGbDeviceController.java`
- `RedisRpcSendRtpController.java`
- `RedisRpcStreamProxyController.java`
- `RedisRpcStreamPushController.java`
- `RedisRpcCloudRecordController.java`
- `RedisRpcDevicePlayController.java`

```java
@Component
@ConditionalOnProperty(name = "wvp.cache.type", havingValue = "redis", matchIfMissing = true)
@RedisRpcController("channelPlay")
public class RedisRpcChannelPlayController extends RpcController {
    // ...
}
```

## 4. 配置开关

### 4.1 添加配置属性

**文件：** `src/main/java/com/genersoft/iot/vmp/conf/WvpCacheProperties.java`

```java
@Configuration
@ConfigurationProperties(prefix = "wvp.cache")
@Data
public class WvpCacheProperties {

    /**
     * 缓存类型：redis或local
     */
    private CacheType type = CacheType.REDIS;

    public enum CacheType {
        REDIS, LOCAL
    }
}
```

### 4.2 配置示例

```yaml
# 集群模式（默认）
wvp:
  cache:
    type: redis
spring:
  redis:
    host: localhost
    port: 6379

# 边缘单机模式
wvp:
  cache:
    type: local
spring:
  redis:
    enabled: false
```

## 5. 改造清单

### 5.1 新增文件

| 文件路径 | 说明 |
|---------|------|
| `LocalCatchStorageImpl.java` | 本地缓存实现 |
| `LocalEventPublisher.java` | 本地事件发布器 |
| `LocalAlarmEventListener.java` | 本地报警事件监听 |
| `LocalStreamChangeEventListener.java` | 本地流变化监听 |
| `LocalGpsEventListener.java` | 本地GPS事件监听 |
| `WvpCacheProperties.java` | 缓存配置属性 |

### 5.2 修改文件

| 文件路径 | 修改内容 |
|---------|---------|
| `RedisRpcConfig.java` | 添加@ConditionalOnProperty |
| `RedisMsgListenConfig.java` | 添加@ConditionalOnProperty |
| `RedisTemplateConfig.java` | 添加@ConditionalOnProperty |
| `PlatformServiceImpl.java` | RedisRpc改为可选依赖 |
| `DeviceServiceImpl.java` | RedisRpc改为可选依赖 |
| `EventPublisher.java` | 添加本地模式支持 |

### 5.3 条件装配的RedisRpc Controller

所有 `RedisRpc*Controller.java` 文件需添加条件装配注解。

## 6. 验证方法

### 6.1 编译测试

```bash
# 编译本地模式
mvn clean package -P local -DskipTests

# 编译集群模式（默认）
mvn clean package -P redis -DskipTests
```

### 6.2 功能测试清单

| 测试项 | 预期结果 |
|-------|---------|
| 启动服务（local模式） | 无Redis依赖，启动成功 |
| 设备注册 | 设备成功上线 |
| 设备点播 | 播放正常 |
| 级联注册 | 向上级注册成功 |
| 目录推送 | 上级收到目录 |
| 位置推送 | 上级收到位置 |
| 报警推送 | 上级收到报警 |
| 服务重启 | 状态需重新建立（预期行为） |

### 6.3 性能对比

| 指标 | Redis模式 | Local模式 |
|-----|----------|----------|
| 内存占用 | 较高 | 低 |
| 启动时间 | 需等待Redis连接 | 无需等待 |
| 重启恢复 | 状态保留 | 状态丢失 |
| 集群扩展 | 支持 | 不支持 |

## 7. 注意事项

### 7.1 数据持久化

本地模式重启后状态会丢失，如果需要持久化可考虑：
- 添加SQLite嵌入式数据库
- 定期将关键状态写入文件

### 7.2 级联限制

- 只支持向上级单一平台级联
- 不支持双向级联（即同时作为上级和下级）

### 7.3 兼容性

- 本地模式无法平滑切换到集群模式
- 切换模式需重启服务

## 8. 附录

### 8.1 Redis用途分析

| Redis Key前缀 | 用途 | 是否可本地化 |
|--------------|------|------------|
| `VMP_DEVICE_INFO` | 设备状态缓存 | ✅ |
| `VMP_GB_INVITE_INFO` | 点播会话信息 | ✅ |
| `VMP_MEDIA_SERVER_INFO` | 媒体服务器信息 | ✅ |
| `VMP_SIP_CSEQ_` | SIP序列号 | ✅ |
| `alarm` | 报警通知 | ❌ 改用Spring事件 |
| `VM_MSG_GPS` | GPS通知 | ❌ 改用Spring事件 |

### 8.2 相关代码路径

- 缓存接口：`src/main/java/com/genersoft/iot/vmp/storager/IRedisCatchStorage.java`
- Redis实现：`src/main/java/com/genersoft/iot/vmp/storager/impl/RedisCatchStorageImpl.java`
- RedisRpc配置：`src/main/java/com/genersoft/iot/vmp/conf/redis/RedisRpcConfig.java`
- 平台服务：`src/main/java/com/genersoft/iot/vmp/gb28181/service/impl/PlatformServiceImpl.java`

## 9. 总结

通过以上改造，WVP-PRO可以在边缘单机环境下运行，无需Redis依赖，同时保留完整的国标级联能力。改造的核心原则是：

1. **保持级联功能**：基于SIP协议的级联不受影响
2. **本地化缓存**：将Redis缓存替换为本地ConcurrentHashMap
3. **配置化开关**：通过配置选择缓存模式
4. **条件装配**：使用Spring条件注解控制组件加载

此改造适用于边缘计算、IoT网关等资源受限场景，可大幅简化部署复杂度。