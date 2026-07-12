# GASShooter 代码阅读指南

> 本文档基于 [GASDocumentation5.3_CN](P:\Others\GASDocumentation5.3_CN\README.md) 和 [GASShooter/README.md](README.md) 的知识体系，结合 GASShooter 工程源码，逐一映射 GAS 核心概念在项目中的具体实现。
>
> ✅ = 已有实现 &nbsp;&nbsp; 📝 = 需扩展 &nbsp;&nbsp; ❓ = 待补充

---

## 目录

1. [项目架构总览](#1-项目架构总览)
2. [AbilitySystemComponent (ASC)](#2-abilitysystemcomponent-asc)
3. [GameplayTags](#3-gameplaytags)
4. [Attributes & AttributeSet](#4-attributes--attributeset)
5. [GameplayEffects](#5-gameplayeffects)
6. [GameplayAbilities](#6-gameplayabilities)
7. [AbilityTasks](#7-abilitytasks)
8. [GameplayCues](#8-gameplaycues)
9. [Targeting (Target Actors & Reticles)](#9-targeting)
10. [Prediction](#10-prediction)
11. [Weapons & Inventory](#11-weapons--inventory)
12. [Interaction System](#12-interaction-system)
13. [常见能力实现](#13-常见能力实现)
14. [调试与优化对照](#14-调试与优化对照)
15. [扩展与补充建议](#15-扩展与补充建议)

---

## 1. 项目架构总览

### 1.1 核心文件结构

```
Source/GASShooter/
├── GASShooter.h                      # 模块入口：输入ID枚举、碰撞通道常量
├── GASShooterGameModeBase.h/cpp      # 游戏模式：死亡→观战→复活流程
│
├── Public/Private
│   ├── Player/
│   │   ├── GSPlayerState.h/cpp       # ASC 所在（PlayerState 模式）
│   │   └── GSPlayerController.h/cpp  # HUD 创建、伤害数字 RPC
│   │
│   ├── Characters/
│   │   ├── GSCharacterBase.h/cpp     # 角色基类：ASC 指针、属性访问
│   │   ├── GSHeroCharacter.h/cpp     # 英雄：相机/背包/武器切换/交互
│   │   ├── GSASCActorBase.h/cpp      # 非Character的ASC持有者基类
│   │   └── GSCharacterMovementComponent.h/cpp  # 冲刺/瞄准/击倒速度修正
│   │
│   │   └── Abilities/                # ⭐ GAS 核心实现
│   │       ├── GSAbilitySystemComponent.h/cpp   # ASC 子类：RPC批处理/多网格蒙太奇/本地GC
│   │       ├── GSAbilitySystemGlobals.h/cpp     # 全局 GAS 设置：自定义 EffectContext 分配
│   │       ├── GSGameplayAbility.h/cpp          # GA 基类：EffectContainer/多网格/成本重载
│   │       ├── GSAbilityTypes.h                 # FGSGameplayEffectContainer 定义
│   │       ├── GSGameplayEffectTypes.h/cpp      # FGSGameplayEffectContext（携带 TargetData）
│   │       ├── GSGameplayCueManager.h/cpp       # GC 管理器（按需加载）
│   │       ├── GSDamageExecutionCalc.h/cpp      # 伤害执行计算（护甲减伤/爆头）
│   │       ├── GSTargetType.h/cpp               # 可蓝图化目标类型
│   │       ├── GSInteractable.h/cpp             # 交互接口
│   │       ├── GSGA_CharacterJump.h/cpp         # 跳跃能力
│   │       ├── GSGATA_Trace.h/cpp               # 可复用追踪 TargetActor（基类）
│   │       ├── GSGATA_LineTrace.h/cpp           # 线条追踪 TargetActor
│   │       ├── GSGATA_SphereTrace.h/cpp         # 球体追踪 TargetActor
│   │       ├── AsyncTaskAttributeChanged.h/cpp  # 蓝图异步：属性变化监听
│   │       └── AsyncTaskGameplayTagAddedRemoved.h/cpp  # 蓝图异步：标签变化监听
│   │
│   │       ├── AttributeSets/
│   │       │   ├── GSAttributeSetBase.h/cpp     # 主属性集（16个属性）
│   │       │   └── GSAmmoAttributeSet.h/cpp     # 弹药属性集（6个属性）
│   │       │
│   │       └── AbilityTasks/
│   │           ├── GSAT_PlayMontageAndWaitForEvent.h/cpp           # 蒙太奇+事件组合
│   │           ├── GSAT_PlayMontageForMeshAndWaitForEvent.h/cpp   # 指定网格蒙太奇
│   │           ├── GSAT_WaitTargetDataUsingActor.h/cpp            # 可复用 TargetActor
│   │           ├── GSAT_WaitInteractableTarget.h/cpp              # 交互对象检测
│   │           ├── GSAT_WaitInputPressWithTags.h/cpp              # 带标签过滤的输入
│   │           ├── GSAT_WaitDelayOneFrame.h/cpp                   # 单帧延迟
│   │           ├── GSAT_WaitChangeFOV.h/cpp                       # FOV 变化等待
│   │           ├── GSAT_MoveSceneCompRelLocation.h/cpp            # 场景组件相对移动
│   │           └── GSAT_ServerWaitForClientTargetData.h/cpp       # 服务端等待客户端目标数据
│   │
│   ├── Weapons/
│   │   ├── GSWeapon.h/cpp              # 武器 Actor：弹药/能力授予/追踪 TargetActor
│   │   └── GSProjectile.h/cpp          # 抛射物
│   │
│   ├── Items/Pickups/
│   │   └── GSPickup.h/cpp              # 拾取物：重叠触发、能力/GE 授予
│   │
│   ├── AI/
│   │   └── GSHeroAIController.h/cpp   # AI 控制器
│   │
│   ├── UI/
│   │   ├── GSHUDWidget.h/cpp           # 主 HUD
│   │   ├── GSHUDReticle.h/cpp          # 准星 Widget
│   │   ├── GSFloatingStatusBarWidget.h/cpp     # 浮动状态条
│   │   └── GSDamageTextWidgetComponent.h/cpp   # 伤害数字组件
│   │
│   └── Characters/Animation/
│       └── GSAnimNotify_PlaySoundForPerspective.h/cpp  # 视角感知音效通知
```

### 1.2 架构分层示意

```
┌─────────────────────────────────────────────────────────────┐
│                     PlayerController                          │
│  (HUD创建, 武器注入, 伤害数字RPC, 复活倒计时)                  │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                      PlayerState                              │
│  ASC ─── Mixed 复制模式                                       │
│  ├─ GSAttributeSetBase      (Health/Mana/Stamina/Shield...)  │
│  └─ GSAmmoAttributeSet      (Rifle/Rocket/Shotgun Reserve)   │
└─────────────────────────┬───────────────────────────────────┘
                          │ InitAbilityActorInfo(PlayerState, HeroCharacter)
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                     HeroCharacter                             │
│  ├─ 第一/三人称相机                                           │
│  ├─ 背包系统 (TArray<AGSWeapon*>)                            │
│  ├─ 武器切换 (预测 + 服务器同步)                              │
│  ├─ 交互系统 (IGSInteractable->Pre/PostInteract)             │
│  └─ 复活/击倒系统                                             │
│                                                                │
│  CharacterMovementComponent (自定义)                           │
│  ├─ 冲刺速度修正                                              │
│  ├─ 瞄准速度修正                                              │
│  └─ 击倒禁止移动                                              │
└─────────────────────────┬───────────────────────────────────┘
                          │  作为 SourceObject 传入
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                       Weapon                                  │
│  ├─ ClipAmmo (Primary/Secondary, 普通浮点数, COND_OwnerOnly) │
│  ├─ 授予武器能力 (GiveAbility, SourceObject = this)          │
│  ├─ 持有 TraceTargetActor (可复用的 GATA_Trace)              │
│  └─ bSourceObjectMustEqualCurrentWeaponToActivate            │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 GASDocumentation vs GASShooter —— ASC 位置对照

| GASDocumentation 描述                 | GASShooter 实现 |
|---------------------------------------|------------------|
| ASC 放在 PlayerState（玩家）/ Pawn（AI） | ✅ `AGSPlayerState` 持有 `UGSAbilitySystemComponent`，AI 通过 `GSHeroAIController` 控制 `AGSHeroCharacter` |
| `IAbilitySystemInterface` 实现         | ✅ `AGSPlayerState`、`AGSCharacterBase`、`AGSWeapon` 均实现 |
| `Mixed` 复制模式                      | ✅ `AbilitySystemComponent->SetReplicatedMode(EGameplayEffectReplicationMode::Mixed)` |
| 服务器初始化 `PossessedBy`，客户端初始化 `OnRep_PlayerState` | ✅ `AGSHeroCharacter::PossessedBy()` + `OnRep_PlayerState()` |
| `NetUpdateFrequency` 提高              | ✅ `AGSPlayerState` 构造函数中设置 `NetUpdateFrequency = 100.f` |

---

## 2. AbilitySystemComponent (ASC)

### 2.1 子类化与扩展

| 知识点（GASDoc）                      | GASShooter 实现                                                                 |
|--------------------------------------|---------------------------------------------------------------------------------|
| 子类化 `UAbilitySystemComponent`      | ✅ `UGSAbilitySystemComponent` 继承自 `UAbilitySystemComponent`                  |
| 重写 `InitAbilityActorInfo`           | ✅ 在 `AGSPlayerState` 中调用                                                   |
| 三种复制模式                          | ✅ `Mixed`（玩家）/ `Minimal`（AI 通过 GameMode 逻辑区分）                       |
| `AbilitySystemGlobals` 子类化         | ✅ `UGSAbilitySystemGlobals`：分配 `FGSGameplayEffectContext`                    |

### 2.2 自定义功能

- **RPC 批处理 (Ability Batching)**：`ShouldDoServerAbilityRPCBatch()` 返回 `true`，`BatchRPCTryActivateAbility()` 使用 `FScopedServerAbilityRPCBatcher` 合并 RPC。
  - 📝 **GASDoc 第 4.6.15 节**：半自动枪将 `CallServerTryActivateAbility` + `ServerSetReplicatedTargetData` + `ServerEndAbility` 合并为一个 RPC；全自动枪合并前两个，`EndAbility` 单独发送。
- **多网格蒙太奇系统**：`FGameplayAbilityLocalAnimMontageForMesh` / `FGameplayAbilityRepAnimMontageForMesh` 数组，支持第一人称和第三人称网格各自播放动画蒙太奇。
- **本地 GameplayCue**：`ExecuteGameplayCueLocal` / `AddGameplayCueLocal` / `RemoveGameplayCueLocal` 绕过复制直接调用 `GameplayCueManager`。
- **预测性 GE 应用**：`BP_ApplyGameplayEffectToSelfWithPrediction` / `BP_ApplyGameplayEffectToTargetWithPrediction`。

### 2.3 待扩展

- ❓ `OnActiveGameplayEffectAddedDelegateToSelf` 在项目中有使用场景吗？当前在代码层未发现广泛注册此委托。

---

## 3. GameplayTags

### 3.1 定义与配置

| 知识点                              | GASShooter 实现                                                    |
|-------------------------------------|-------------------------------------------------------------------|
| `DefaultGameplayTags.ini` 定义       | ✅ `Config/DefaultGameplayTags.ini` — 共 77 个标签                   |
| 标签命名层次                         | ✅ `Ability.*`, `State.*`, `Weapon.*`, `Effect.*`, `GameplayCue.*`, `Data.*`, `Event.*` |
| Fast Replication                     | ✅ 在项目设置中启用                                                |

### 3.2 核心标签分类

| 类别              | 示例标签                                                                 |
|-------------------|-------------------------------------------------------------------------|
| Ability           | `Ability.Jump`, `Ability.Sprint`, `Ability.Interact`, `Ability.Weapon.PrimaryFire` |
| State             | `State.Dead`, `State.KnockedDown`, `State.BleedingOut`, `State.Sprinting`, `State.Interacting` |
| Weapon            | `Weapon.Equipped.Rifle`, `Weapon.Ammo.Rifle`, `Weapon.IsFiring`, `Weapon.Activating` |
| Effect            | `Effect.Damage.CanHeadShot`, `Effect.Damage.HeadShot`, `Effect.RemoveOnDeath` |
| GameplayCue       | `GameplayCue.Weapon.Rifle.Fire`, `GameplayCue.Weapon.Shotgun.Fire`      |
| Data              | `Data.Damage`, `Data.ReloadAmount`                                      |
| Event             | `Event.EndAbility`                                                      |

### 3.3 标签响应

| 知识点                            | GASShooter 实现                                                                 |
|-----------------------------------|---------------------------------------------------------------------------------|
| `RegisterGameplayTagEvent`        | ✅ `AsyncTaskGameplayTagAddedRemoved` 封装                                      |
| `LooseGameplayTags`               | ✅ `State.Dead` 使用 `AddLooseGameplayTag` / `RemoveLooseGameplayTag` 立即响应   |
| 标签阻止能力激活                   | ✅ `ActivationBlockedTags` 中默认包含 `State.Dead` 和 `State.KnockedDown`         |
| `HasMatchingGameplayTag`           | ✅ `PreReplication()` 中检查 `Weapon.IsFiring` 以抑制弹药复制                     |

### 3.4 待扩展
- ❓ 缺乏 `Activation.Fail.*` 系列标签（`Activation.Fail.BlockedByTags` 等），当前未配置 `DefaultGame.ini` 中的失败标签映射。
- 📝 可增加 `Cooldown` 相关的标签（如 `Cooldown.Weapon`）以支持通用冷却查询。

---

## 4. Attributes & AttributeSet

### 4.1 主属性集 (`GSAttributeSetBase`)

| 属性名                | 复制 | Meta Attribute | 说明                           |
|-----------------------|------|----------------|--------------------------------|
| Health                | ✅   |                | 生命值                         |
| MaxHealth             | ✅   |                | 最大生命值                     |
| HealthRegenRate       | ✅   |                | 生命恢复速率                   |
| Mana                  | ✅   |                | 法力值                         |
| MaxMana               | ✅   |                | 最大法力值                     |
| ManaRegenRate         | ✅   |                | 法力恢复速率                   |
| Stamina               | ✅   |                | 耐力值                         |
| MaxStamina            | ✅   |                | 最大耐力值                     |
| StaminaRegenRate      | ✅   |                | 耐力恢复速率                   |
| Shield                | ✅   |                | 护盾值                         |
| MaxShield             | ✅   |                | 最大护盾值                     |
| ShieldRegenRate       | ✅   |                | 护盾恢复速率                   |
| Armor                 | ✅   |                | 护甲（伤害减免）               |
| **Damage**            | ❌   | ✅ Meta        | 伤害占位符（`PostGameplayEffectExecute` 中消费） |
| MoveSpeed             | ✅   |                | 移动速度                       |
| CharacterLevel        | ✅   |                | 角色等级                       |
| XP                    | ✅   |                | 经验值                         |
| XPBounty              | ✅   |                | 击杀经验奖励                   |
| Gold                  | ✅   |                | 金币                           |
| GoldBounty            | ✅   |                | 击杀金币奖励                   |

### 4.2 弹药属性集 (`GSAmmoAttributeSet`)

| 属性名                      | 复制 | 说明                    |
|-----------------------------|------|-------------------------|
| RifleReserveAmmo            | ✅   | 步枪备用弹药            |
| MaxRifleReserveAmmo         | ✅   | 最大步枪备用弹药        |
| RocketReserveAmmo           | ✅   | 火箭筒备用弹药          |
| MaxRocketReserveAmmo        | ✅   | 最大火箭筒备用弹药      |
| ShotgunReserveAmmo          | ✅   | 霰弹枪备用弹药          |
| MaxShotgunReserveAmmo       | ✅   | 最大霰弹枪备用弹药      |

> 静态工具函数：`GetReserveAmmoAttributeFromTag(FGameplayTag)` 和 `GetMaxReserveAmmoAttributeFromTag(FGameplayTag)` 将 `Weapon.Ammo.Rifle` / `Rocket` / `Shotgun` 标签映射到对应属性。

### 4.3 GASDoc 知识点映射

| 知识点                                     | GASShooter 实现                                                                 |
|--------------------------------------------|---------------------------------------------------------------------------------|
| `PreAttributeChange()` 限制 CurrentValue   | ✅ 所有属性限制到对应的 Max 属性值（如 Health ≤ MaxHealth），MoveSpeed 限制 [150, 1000] |
| `PostGameplayEffectExecute()` 处理伤害     | ✅ `Damage` Meta Attribute → 先减 Shield → 再减 Health；触发死亡/击倒广播          |
| `BaseValue` vs `CurrentValue`              | ✅ 严格遵守：Instant GE 修改 BaseValue，Duration/Infinite GE 修改 CurrentValue      |
| `ATTRIBUTE_ACCESSORS` 宏                   | ✅ 所有属性都使用了                                                               |
| `GAMEPLAYATTRIBUTE_REPNOTIFY`              | ✅ 每个复制属性都有 `OnRep_*` 实现                                                 |
| `OnAttributeAggregatorCreated`             | ❓ 未重写                                                                         |
| Derived Attributes (派生属性)              | ❓ 未实现                                                                         |

### 4.4 待扩展

- 📝 当前 `Shield` 的手动消耗在 `PostGameplayEffectExecute` 中，可以考虑使用 `Infinite GE` + `ShieldRegenRate` 机制实现随时间自动恢复。
- ❓ `OnAttributeAggregatorCreated` 未重写，如果后续需要"只应用最弱的减速效果"（类似 Paragon），需要补充。
- 📝 可考虑添加 `StaminaRegenDelay` 属性，实现耐力消耗后延迟恢复。

---

## 5. GameplayEffects

### 5.1 GE 应用与移除

| 知识点                           | GASShooter 实现                                          |
|----------------------------------|----------------------------------------------------------|
| `ApplyGameplayEffectToSelf`      | ✅ `InitializeAttributes()` 中使用 Instant GE 初始化属性  |
| `ApplyGameplayEffectSpecToSelf`  | ✅ `GSDamageExecutionCalc` 中 Apply 伤害 GE                |
| `MakeOutgoingGameplayEffectSpec` | ✅ `UGSGameplayAbility::MakeEffectContainerSpec` 中使用    |
| `RemoveActiveGameplayEffect`     | ✅ `AGSWeapon::RemoveAbilities()` 中移除武器 GE            |
| 监听 GE 添加/移除                | ✅ `AsyncTaskEffectStackChanged` 监听堆叠变化              |

### 5.2 自定义 GameplayEffectContext

| 知识点                            | GASShooter 实现                                                                 |
|-----------------------------------|---------------------------------------------------------------------------------|
| 子类化 `FGameplayEffectContext`   | ✅ `FGSGameplayEffectContext` 继承自 `FGameplayEffectContext`                     |
| 添加 `TargetData`                 | ✅ `FGameplayAbilityTargetDataHandle TargetData` 成员                             |
| 重写 `GetScriptStruct()`          | ✅ 返回 `FGSGameplayEffectContext::StaticStruct()`                                |
| 重写 `Duplicate()`                | ✅ 深拷贝 Actor、HitResult，浅拷贝 TargetData                                      |
| 重写 `NetSerialize()`             | ✅ 在 `.cpp` 中实现                                                              |
| `AllocGameplayEffectContext()`    | ✅ `UGSAbilitySystemGlobals::AllocGameplayEffectContext()` 返回新的子类实例        |
| 在 GameplayCue 中访问             | ✅ 霰弹枪的多个击中结果通过 `EffectContext` 传递给 GC                               |

### 5.3 伤害执行计算 (`GSDamageExecutionCalc`)

| 知识点                                   | GASShooter 实现                                                                 |
|------------------------------------------|---------------------------------------------------------------------------------|
| `GameplayEffectExecutionCalculation`      | ✅ `UGSDamageExecutionCalc`                                                      |
| 捕获 Source 属性 (Snapshot)               | ✅ `Damage` 属性从 Source 快照捕获                                                |
| 捕获 Target 属性 (No Snapshot)            | ✅ `Armor` 属性从 Target 非快照捕获                                               |
| 护甲减伤公式                              | ✅ `MitigatedDamage = BaseDamage * (100 / (100 + Armor))`                        |
| 爆头检测                                  | ✅ 检查 `Effect.Damage.CanHeadShot` 标签 + HitResult 骨骼名 `b_head` → 1.5x 伤害 |
| `SetByCaller` 读取                        | ✅ 支持 `Data.Damage` 标签的 SetByCaller 值                                       |
| `EffectContext` 读取                      | ✅ 访问 `FGSGameplayEffectContext::GetTargetData()` 获取 HitResult                |

### 5.4 待扩展

- 📝 缺少 **生命偷取 (Lifesteal)** 实现（GASDoc 5.4 节）——可在 `GSDamageExecutionCalc` 中根据 `Effect.CanLifesteal` 标签动态创建 GE。
- 📝 缺少 **暴击 (Critical Hit)** 实现（GASDoc 5.6 节）——可在 `GSDamageExecutionCalc` 中根据 `Effect.CanCrit` 标签和 Source 属性计算暴击。
- ❓ 当前项目未使用 `CustomApplicationRequirement`（GASDoc 4.5.13 节），无明显需求但可作为扩展点。
- 📝 `GameplayEffectContainer`（GASDoc 4.5.18 节 / 8.1 节）在项目中命名为 `FGSGameplayEffectContainer`，已完整实现但文档未充分说明其 targeting 集成方式。

---

## 6. GameplayAbilities

### 6.1 基类 (`UGSGameplayAbility`)

| 知识点                                  | GASShooter 实现                                                                 |
|-----------------------------------------|---------------------------------------------------------------------------------|
| `InstancingPolicy`                      | ✅ `EGameplayAbilityInstancingPolicy::InstancedPerActor`（默认）                 |
| `NetExecutionPolicy`                     | ✅ 混合使用 `LocalPredicted`（Sprint）/ `ServerOnly`（被动）/ `LocalOnly`（输入管理） |
| `Ability Tags` / `ActivationBlockedTags` | ✅ `State.Dead` + `State.KnockedDown` 默认阻塞所有能力                           |
| `bActivateAbilityOnGranted`             | ✅ 被动能力授予时自动激活                                                        |
| `OnAvatarSet`                           | ✅ 检查 `bActivateAbilityOnGranted` → 调用 `TryActivateAbility`                  |
| `CommitAbility` / `CheckCost`           | ✅ 重写 `GSCheckCost` / `GSApplyCost` 支持自定义弹药消耗                          |
| 传入绑定输入                             | ✅ `EGSAbilityInputID` 枚举 + `FGameplayAbilityInputBinds`                       |

### 6.2 武器能力体系

| 武器          | 主要能力 (LMB)                     | 次要能力 (RMB)                | 交替能力 (MMB)           |
|---------------|------------------------------------|-------------------------------|--------------------------|
| 步枪 (Rifle)  | 命中扫描射击 (根据射速模式)         | 瞄准 (减小散布)               | 切换射速模式 (全自动/半自动/三连发) |
| 火箭发射器    | 发射火箭                           | 瞄准 + 锁定目标(追踪火箭)     | ❌ 无                    |
| 霰弹枪        | 命中扫描散射 (根据射速模式)         | 瞄准 (减小散射)                | 切换射速模式 (半自动/全自动)    |

> **关键设计**：所有武器能力使用同一个 `GA_WeaponFire` 蓝图类，通过 `SourceObject` 区分具体武器，通过 `bSourceObjectMustEqualCurrentWeaponToActivate` 确保只有装备的武器可激活。

### 6.3 RPC 批处理 (Ability Batching)

| GASDoc 知识点               | GASShooter 实现                                                                 |
|----------------------------|---------------------------------------------------------------------------------|
| `ShouldDoServerAbilityRPCBatch` | ✅ `UGSAbilitySystemComponent::ShouldDoServerAbilityRPCBatch()` 返回 `true`       |
| `FScopedServerAbilityRPCBatcher` | ✅ `BatchRPCTryActivateAbility()` 中使用                                           |
| 半自动 = 合并 3 个 RPC      | ✅ 激活 + TargetData + EndAbility 在一个批处理中                                   |
| 全自动 = 合并前 2 个 RPC    | ✅ 激活 + 第一发 TargetData 批处理，后续每发独立 TargetData，最后 EndAbility 单独  |

### 6.4 弹药成本系统

| GASDoc 知识点                     | GASShooter 实现                                                                 |
|-----------------------------------|---------------------------------------------------------------------------------|
| 物品属性实现（4.4.2.3 节）         | ✅ **ClipAmmo** = 武器上的普通浮点数（`COND_OwnerOnly` 复制）<br>✅ **ReserveAmmo** = PlayerState 上的 `AmmoAttributeSet` |
| `PreReplication` 抑制复制          | ✅ `Weapon.IsFiring` 标签激活时抑制 `PrimaryClipAmmo` / `SecondaryClipAmmo` 复制   |
| 重写 `CheckCost` / `ApplyCost`    | ✅ `GSCheckCost`（BlueprintNativeEvent）+ `GSApplyCost`（BlueprintNativeEvent）    |
| Cost GE 方式                      | ❌ 不使用 Cost GE，使用蓝图自定义成本逻辑                                           |

### 6.5 冷却系统

| GASDoc 知识点                           | GASShooter 实现                                          |
|------------------------------------------|----------------------------------------------------------|
| Cooldown GE (Duration)                  | ✅ 每个武器能力有对应的冷却 GE                              |
| Cooldown Tag                            | ✅ 使用 `Ability.Cooldown.Weapon` 等标签                   |
| 预测冷却                                | ❓ 未实现真正的预测冷却（遵循 GASDoc 4.5.15.3 节的限制说明） |

### 6.6 待扩展

- 📝 缺少 **共享冷却槽位** 的通用设计（当前每个能力有自己的冷却 GE）。
- 📝 缺少 `ActivationFailedTags` 配置（GASDoc 4.6.4.2 节）——需要在 `DefaultGame.ini` 和标签列表中补充。
- ❓ 未实现 `GameplayAbilitySet`（GASDoc 4.6.14 节）——当前在 `AGSCharacterBase` 中以 `TArray<TSubclassOf<UGSGameplayAbility>>` 管理，数据资产模式可作为扩展。
- 📝 武器能力体系和射速模式管理分布在蓝图中，C++ 侧文档不足。

---

## 7. AbilityTasks

### 7.1 自定义 AbilityTasks 清单

| 名称                                    | GASDoc 知识点               | 用途                                                                 |
|-----------------------------------------|----------------------------|----------------------------------------------------------------------|
| `GSAT_PlayMontageAndWaitForEvent`       | 4.7.2 节 - 自定义 AT       | 蒙太奇播放 + 等待动画通知事件                                        |
| `GSAT_PlayMontageForMeshAndWaitForEvent`| 4.7.2 节 - 多网格          | 在指定骨骼网格上播放蒙太奇 + 等待事件                                |
| `GSAT_WaitTargetDataUsingActor`         | 4.11.2 节 - 可复用 TA      | 使用已有的 TargetActor 等待目标数据（不销毁）                         |
| `GSAT_WaitInteractableTarget`           | 5.9 节 - 交互系统          | 定时线条追踪检测可交互对象                                           |
| `GSAT_WaitInputPressWithTags`           | 4.6.2 节 - 输入绑定        | 带标签过滤的输入等待                                                 |
| `GSAT_WaitDelayOneFrame`                | —                          | 单帧延迟                                                             |
| `GSAT_WaitChangeFOV`                    | —                          | 等待 FOV 变化完成                                                    |
| `GSAT_MoveSceneCompRelLocation`         | —                          | 场景组件的相对位置移动                                               |
| `GSAT_ServerWaitForClientTargetData`    | 4.11.2 节 - 服务端验证     | 服务端等待客户端发送的 TargetData                                    |

### 7.2 GASDoc 知识点映射

| 知识点                              | GASShooter 实现                                          |
|-------------------------------------|---------------------------------------------------------|
| `PlayMontageAndWait` 基础          | ✅ `GSAT_PlayMontageAndWaitForEvent` 扩展版              |
| `WaitGameplayEvent`                | ✅ 与蒙太奇组合在一个 Task 中                              |
| `WaitTargetData`                   | ✅ `GSAT_WaitTargetDataUsingActor`（可复用而非每次生成）  |
| `WaitInteractableTarget`           | ✅ 交互系统核心                                          |
| `AbilityTask` 组成模式             | ✅ 静态创建函数 + 输出委托 + Activate + OnDestroy        |
| `bSimulatedTask` 模拟              | ❓ 未使用                                               |
| `bTickingTask` Tick 能力          | ❓ 未使用                                               |

---

## 8. GameplayCues

### 8.1 类型与使用

| GASDoc 知识点                         | GASShooter 实现                                       |
|---------------------------------------|------------------------------------------------------|
| `GameplayCueNotify_Static` (Execute)  | ✅ 武器开火、火箭弹命中效果                              |
| `GameplayCueNotify_Actor` (Add/Remove)| ✅ 冲刺、击倒状态效果                                   |
| 标签必须以 `GameplayCue.` 开头        | ✅ `GameplayCue.Weapon.Rifle.Fire` 等                  |
| 从 GE 触发 GC                        | ✅ 在武器能力 GE 中设置 GC 标签                         |
| 从 GA 触发 GC                        | ✅ `UGSAbilitySystemComponent::ExecuteGameplayCue`     |
| 本地 GameplayCue                     | ✅ `ExecuteGameplayCueLocal` 等方法                    |
| `ShouldAsyncLoadRuntimeObjectLibraries` | ✅ `UGSGameplayCueManager::ShouldAsyncLoadRuntimeObjectLibraries` 返回 `false`（按需加载） |
| `GameplayCueNotifyPaths`             | ✅ `DefaultGame.ini` 中配置路径                         |

### 8.2 已定义的 GameplayCue 标签

| 标签                                          | 类型          | 触发时机       |
|-----------------------------------------------|--------------|----------------|
| `GameplayCue.Hero.KnockedDown`                | Actor        | 击倒时添加     |
| `GameplayCue.Hero.Revived`                    | Static       | 复活时执行     |
| `GameplayCue.Weapon.Rifle.Fire`               | Static       | 开火时执行     |
| `GameplayCue.Weapon.RocketLauncher.Fire`      | Static       | 火箭发射时执行 |
| `GameplayCue.Weapon.RocketLauncher.Impact`    | Static       | 火箭命中时执行 |
| `GameplayCue.Weapon.Shotgun.Fire`             | Static       | 霰弹枪开火时执行 |
| `GameplayCue.Ability.Sprinting`               | Actor        | 冲刺时添加     |

### 8.3 待扩展

- 📝 **GC 批处理**（GASDoc 7.2 节 / 4.8.7 节）：霰弹枪的 8 个弹丸击中效果当前通过 `EffectContext` 的 TargetData 批量传递，但未使用 `FScopedGameplayCueSendContext` 进一步优化 RPC。可参考 GASDoc 4.8.7.1 的手动 RPC 批处理方案。
- 📝 可增加 `GameplayCue.Hero.ShieldHit`、`GameplayCue.Hero.SprintEnd` 等标签。

---

## 9. Targeting

### 9.1 Target Actors

| GASDoc 知识点                          | GASShooter 实现                                                                 |
|----------------------------------------|---------------------------------------------------------------------------------|
| 基础 TargetActor                       | ✅ `AGSGATA_Trace`（基类）：可复用、可配置散布/精度                               |
| `WaitTargetData` AbilityTask           | ✅ `GSAT_WaitTargetDataUsingActor`（复用已有 TargetActor，不销毁）                 |
| `Instant` 确认（不需要用户输入）        | ✅ 步枪/霰弹枪命中扫描        |
| `UserConfirmed` 确认                   | ✅ 火箭发射器追踪火箭锁定                                                         |
| `Custom` 确认                          | ❓ 未使用                                                                         |
| 持久 Hit Result                        | ✅ `AGSGATA_Trace` 支持配置 `bPersistentHitResults`，火箭筒锁定场景使用            |
| 散布/精度系统                          | ✅ `BaseSpread`, `AimingSpreadMod`, `TargetingSpreadIncrement`, `TargetingSpreadMax` |
| `ShouldProduceTargetDataOnServer`      | ❓ 未实现（当前始终从客户端发送 TargetData 到服务器）                              |

### 9.2 GameplayAbilityWorldReticles

| GASDoc 知识点                          | GASShooter 实现                                                                 |
|----------------------------------------|---------------------------------------------------------------------------------|
| Reticle 基本概念                        | ✅ `AGameplayAbilityWorldReticle`：火箭筒锁定目标时显示红色指示器                  |
| WidgetComponent 实现                    | ✅ 通过 Widget Component 在屏幕空间显示 UMG Widget                                 |
| `OnValidTargetChanged`                  | ✅ 目标有效/无效时切换准星状态                                                    |
| 持久 Reticle（锁定后保持）              | ✅ 火箭筒追踪能力中，锁定目标后 Reticle 持续显示                                  |

### 9.3 待扩展

- 📝 `ShouldProduceTargetDataOnServer` 的补充——服务器端验证 TargetData 可以防止作弊，适合火箭筒锁定等场景。
- 📝 自定义 `FGameplayTargetDataFilter` 子类（GASDoc 4.11.3 节）——当前使用内置过滤器，无高级过滤需求但可作为扩展点。
- 📝 `GameplayEffectContainer` 中的 targeting（GASDoc 4.11.5 节）——`UGSTargetType` 提供了蓝图化的目标选择，但文档中对 `UseOwner` 和 `UseEventData` 的实现说明不足。

---

## 10. Prediction

### 10.1 已预测的内容

| GASDoc 可预测内容        | GASShooter 实现                                                               |
|--------------------------|------------------------------------------------------------------------------|
| 能力激活                 | ✅ `LocalPredicted` 策略的能力（如冲刺、武器射击）                              |
| 属性修改 (Modifier)      | ✅ 冲刺耐力消耗（通过 `WaitNetSync` + `OnlyServerWait` 创建新预测窗口）        |
| GameplayTag 修改         | ✅ 冲刺时添加 `State.Sprinting` 标签                                           |
| GameplayCue 事件         | ✅ `LocalPredicted` 能力中的 GC                                                |
| 动画蒙太奇               | ✅ 武器开火动画预测播放                                                        |
| 移动 (CMC)              | ✅ 冲刺/瞄准速度变化通过 `UGSCharacterMovementComponent` 和 `FGSSavedMove` 预测 |

### 10.2 未预测的内容

| GASDoc 不可预测内容          | GASShooter 处理方式                                              |
|-----------------------------|------------------------------------------------------------------|
| GameplayEffect 移除         | ❌ 不预测，等待服务器复制                                          |
| 周期性效果 (Periodic)       | ❌ 不使用周期性效果做关键逻辑                                      |
| `ExecutionCalculations`     | ❌ 伤害通过 `GSDamageExecutionCalc`（不可预测），仅在服务端执行      |
| 弹丸 Actor 生成             | ❌ 不预测抛射物生成（GASShooter 明确说明不预测）                    |

### 10.3 预测武器切换

| GASDoc 知识点          | GASShooter 实现                                                               |
|------------------------|------------------------------------------------------------------------------|
| 预测性改变 `CurrentWeapon` | ✅ 自主客户端立即切换武器，通过 `ServerSyncCurrentWeapon` / `ClientSyncCurrentWeapon` 同步 |
| 抑制复制               | ✅ `CurrentWeapon` 仅复制到模拟客户端 (simulated proxy)                         |

### 10.4 待扩展

- 📝 GASDoc 4.10.3 节：预测性生成 Actor 的实现——当前项目不预测抛射物，但如果需要追踪火箭的预测，可以参考 Unreal Tournament 的假抛射物方法。
- 📝 GASDoc 4.10.1 ~ 4.10.2 节：项目中 `WaitNetSync` + `OnlyServerWait` 的耐力消耗模式是一个很好的 Scoped Prediction Window 示例，但在文档中未突出说明。

---

## 11. Weapons & Inventory

### 11.1 武器实现 (`AGSWeapon`)

| 结构              | 说明                                                                 |
|-------------------|----------------------------------------------------------------------|
| PrimaryClipAmmo   | 弹匣主弹药，普通 float，`COND_OwnerOnly` 复制                         |
| SecondaryClipAmmo | 弹匣副弹药（未使用，预留给枪榴弹等）                                  |
| 弹药复制抑制       | `PreReplication()` 中检查 `Weapon.IsFiring` 标签                       |
| 武器能力授予       | `AddAbilities()` / `RemoveAbilities()` 管理 GA 生命周期               |
| TraceTargetActor   | 持有可复用的 `AGSGATA_Trace` 子类实例                                  |
| bSourceObject绑定  | 能力通过 `SourceObject == ThisWeapon` 检查是否可用                     |

### 11.2 背包系统

```
FGSHeroInventory
├── TArray<AGSWeapon*> Weapons          # 当前背包武器列表
├── int32 CurrentWeaponIndex            # 当前武器索引
└── AGSWeapon* CurrentWeapon            # 当前武器指针
```

| 操作       | 实现方法                                                                 |
|------------|--------------------------------------------------------------------------|
| 添加武器   | `AddWeapon()` → 添加至数组 → `GiveAbilities()`                           |
| 移除武器   | `RemoveWeapon()` → `RemoveAbilities()` → 从数组移除                       |
| 切换武器   | `SwitchWeapon(int32 Index)` / `NextWeapon()` / `PrevWeapon()`             |
| 预测切换   | 自主客户端立即切换 → `ServerSyncCurrentWeapon` RPC                       |

### 11.3 待扩展

- 📝 `SecondaryClipAmmo` 当前未使用——GASShooter 作者在 README 中提到其设计用途是"枪榴弹等"，可实现二级弹药能力。
- 📝 当背包切换武器时，对 `PreReplication` 中被抑制的弹药属性同步机制缺少文档说明。
- ❓ 当前武器系统不支持从地面拾取同类型武器并合并弹药。

---

## 12. Interaction System

### 12.1 接口与实现

| GASDoc 知识点             | GASShooter 实现                                                                 |
|---------------------------|--------------------------------------------------------------------------------|
| 5.9 节 - 一键交互系统      | ✅ `IGSInteractable` 接口 + `GSAT_WaitInteractableTarget`                      |
| `PreInteract` / `PostInteract` | ✅ 两阶段交互：PreInteract 播动画 → PostInteract 应用效果                      |
| 同步类型                   | ✅ `EInteractionSyncType`：`OnlyClientWait`（复活等待服务器）、`OnlyServerWait`（无需等待） |
| 取消交互                   | ✅ `InteractableCancelInteraction()` / `ServerCancelInteraction()`              |
| 交互提示 UI               | ✅ `GSHUDWidget` 显示交互提示文字和进度条                                       |

### 12.2 可交互对象

| 对象          | 交互行为                               | 实现方式               |
|---------------|----------------------------------------|------------------------|
| 倒地队友      | 按 E 复活（需按住以完成引导）           | `PreInteract` → 复活动画 → `PostInteract` → 应用复活 GE |
| 武器箱        | 按 E 打开获取武器                       | 打开动画 → 生成武器并加入背包     |
| 滑动门        | 按 E 开关门                             | 直接触发门的开关逻辑               |

### 12.3 待扩展

- 📝 当前交互缺少 **长按 vs 短按** 的区分（GASShooter README 提到"Press or Hold 'E'"，但源码中 `GSAT_WaitInteractableTarget` 主要是定时检测，长按引导逻辑在蓝图中）。
- 📝 可增加交互对象的优先级系统（同时面对多个交互对象时选择哪个）。

---

## 13. 常见能力实现

### 13.1 跳跃

| GASDoc 知识点     | GASShooter 实现                                                                 |
|-------------------|---------------------------------------------------------------------------------|
| 非实例化能力       | ✅ `GSGA_CharacterJump` -> `Non-Instanced` / `LocalPredicted`                    |
| 简单激活          | ✅ `ActivateAbility()` 中调用 `Character->Jump()`                               |

### 13.2 冲刺

| GASDoc 知识点               | GASShooter 实现                                                                 |
|-----------------------------|---------------------------------------------------------------------------------|
| 5.2 节 - Sprint             | ✅ 蓝图 `GA_Sprint_BP` + `UGSCharacterMovementComponent`                         |
| 输入响应                    | ✅ `Left Shift` → `Ability.Sprint`                                              |
| 耐力消耗                    | ✅ `WaitNetSync(OnlyServerWait)` 创建预测窗口 → `ApplyCost` 消耗耐力              |
| 速度修改                    | ✅ `GetMaxSpeed()` 检查 `State.Sprinting` 标签                                    |
| 预测                        | ✅ `LocalPredicted`                                                              |

### 13.3 瞄准 (ADS)

| GASDoc 知识点               | GASShooter 实现                                                                 |
|-----------------------------|---------------------------------------------------------------------------------|
| 5.3 节 - Aim Down Sights   | ✅ 蓝图 `GA_AimDownSight_BP`                                                     |
| 速度减少                    | ✅ `GetMaxSpeed()` 检查 `Weapon.Aiming` 标签                                     |
| FOV 变化                    | ✅ `GSAT_WaitChangeFOV` 等待相机 FOV 变化完成                                    |

### 13.4 击倒与复活

| 阶段     | 逻辑                                                                 |
|----------|----------------------------------------------------------------------|
| 击倒     | HP 归零 → `State.KnockedDown` 标签 → 取消所有能力 → 进入可复活状态     |
| 复活     | 队友交互 → `State.KnockedDown` 移除 → 恢复 HP → `GameplayCue.Hero.Revived` |
| 死亡     | 击倒状态下 HP 归零 → `State.Dead` 标签 → `FinishDying()` → 观战 → 5s 后复活 |

### 13.5 待扩展

- 📝 **Stun 效果**（GASDoc 5.1 节）：当前 `State.KnockedDown` 实现了类似眩晕的效果（取消能力 + 阻塞激活 + 禁止移动），但缺少一个独立的 `State.Debuff.Stun` 标签体系。
- 📝 **生命偷取**（GASDoc 5.4 节）：可在 `GSDamageExecutionCalc` 中扩展。
- 📝 **暴击**（GASDoc 5.6 节）：可在 `GSDamageExecutionCalc` 中扩展。
- 📝 **非堆叠减速**（GASDoc 5.7 节）：可通过 `OnAttributeAggregatorCreated` + `MostNegativeMod_AllPositiveMods` 实现。

---

## 14. 调试与优化对照

### 14.1 调试

| GASDoc 知识点                 | GASShooter 现状                                        |
|-------------------------------|-------------------------------------------------------|
| 6.1 - `showdebug abilitysystem` | ✅ 支持，需 GameMode 设置 HUD 类                        |
| 6.2 - Gameplay Debugger        | ✅ 支持                                                |
| 6.3 - GAS Logging              | ✅ 标准 GAS 日志                                       |
| 9.1 - ASC 初始化错误           | ✅ 已在 PossessedBy + OnRep_PlayerState 中初始化        |
| 9.2 - ScriptStructCache        | ✅ 已在 `GSEngineSubsystem` 中调用 `InitGlobalData()`   |

### 14.2 优化

| GASDoc 知识点             | GASShooter 实现                                           |
|---------------------------|----------------------------------------------------------|
| 7.1 - Ability Batching    | ✅ `UGSAbilitySystemComponent` 全支持                       |
| 7.2 - Gameplay Cue Batching | 📝 部分实现（EffectContext 携带 TargetData），可进一步优化 |
| 7.3 - ASC 复制模式        | ✅ `Mixed`（玩家）/ `Minimal`（AI）                         |
| 7.4 - Attribute Proxy Replication | ❓ 未实现（小规模项目不需要）                           |
| 7.5 - ASC Lazy Loading    | ❓ 未实现（小规模项目不需要）                                |

---

## 15. 扩展与补充建议

以下内容为 GASDocumentation 描述但 GASShooter 未实现或需要补充的知识点，按优先级排列：

### 🔴 高优先级（推荐实现）

| 编号 | 知识点 | GASDoc 章节 | 说明 |
|------|--------|-------------|------|
| 1 | `ActivationFailedTags` 配置 | 4.6.4.2 | 补充 `DefaultGame.ini` 和标签列表，方便调试能力激活失败原因 |
| 2 | 生命偷取 (Lifesteal) | 5.4 | 在 `GSDamageExecutionCalc` 中检查 `Effect.CanLifesteal` 标签，动态创建生命恢复 GE |
| 3 | 暴击 (Critical Hit) | 5.6 | 在 `GSDamageExecutionCalc` 中检查 `Effect.CanCrit` 标签，根据 Source 属性计算暴击倍率 |

### 🟡 中优先级（推荐了解）

| 编号 | 知识点 | GASDoc 章节 | 说明 |
|------|--------|-------------|------|
| 4 | `OnAttributeAggregatorCreated` | 4.4.7 | 实现"只应用最弱减速"效果 |
| 5 | `CustomApplicationRequirement` | 4.5.13 | 复杂 GE 条件检查的扩展入口 |
| 6 | `GameplayAbilitySet` 数据资产 | 4.6.14 | 将角色能力配置移到 DataAsset 中 |
| 7 | `ShouldProduceTargetDataOnServer` | 4.11.2 | 服务端生成 TargetData 用于作弊防护 |
| 8 | 主动技能 (Meteor-like) | 2 / 5.1 | 实现类似 GASDocumentation 示例项目的流星技能（Area-of-Effect 瞄准→伤害+眩晕） |
| 9 | 派生属性 (Derived Attributes) | 4.3.5 | 通过 Infinite GE + MMC 实现属性自动计算（如从等级派生生命值上限） |

### 🟢 低优先级（进阶优化）

| 编号 | 知识点 | GASDoc 章节 | 说明 |
|------|--------|-------------|------|
| 10 | GC 批处理 (FScopedGameplayCueSendContext) | 4.8.7 / 7.2 | 霰弹枪等多弹丸场景进一步优化 RPC |
| 11 | Attribute Proxy Replication | 7.4 | 大规模多人游戏的属性复制优化 |
| 12 | ASC Lazy Loading | 7.5 | 大量可破坏对象的 ASC 惰性加载 |
| 13 | 预测性生成 Actor | 4.10.3 | 如假抛射物预测火箭弹轨迹 |
| 14 | 自定义 `FGameplayTargetDataFilter` | 4.11.3 | 更精细的目标过滤 |
| 15 | 共享冷却槽位系统 | 4.5.15 | 通用冷却 GE + SetByCaller + MMC 设计 |

---

## 附录：文件路径速查表

| 知识点 | 核心文件路径 |
|--------|-------------|
| ASC | `Source/GASShooter/Public/Characters/Abilities/GSAbilitySystemComponent.h/.cpp` |
| AttributeSet (主) | `Source/GASShooter/Public/Characters/Abilities/AttributeSets/GSAttributeSetBase.h/.cpp` |
| AttributeSet (弹药) | `Source/GASShooter/Public/Characters/Abilities/AttributeSets/GSAmmoAttributeSet.h/.cpp` |
| GA 基类 | `Source/GASShooter/Public/Characters/Abilities/GSGameplayAbility.h/.cpp` |
| EffectContainer | `Source/GASShooter/Public/Characters/Abilities/GSAbilityTypes.h` |
| EffectContext | `Source/GASShooter/Public/Characters/Abilities/GSGameplayEffectTypes.h/.cpp` |
| 伤害计算 | `Source/GASShooter/Public/Characters/Abilities/GSDamageExecutionCalc.h/.cpp` |
| TargetActor (Trace) | `Source/GASShooter/Public/Characters/Abilities/GSGATA_Trace.h/.cpp` |
| AbilityTasks | `Source/GASShooter/Public/Characters/Abilities/AbilityTasks/*` |
| 武器 | `Source/GASShooter/Public/Weapons/GSWeapon.h/.cpp` |
| 交互接口 | `Source/GASShooter/Public/Characters/Abilities/GSInteractable.h/.cpp` |
| PlayerState | `Source/GASShooter/Public/Player/GSPlayerState.h/.cpp` |
| 角色基类 | `Source/GASShooter/Public/Characters/GSCharacterBase.h/.cpp` |
| 英雄角色 | `Source/GASShooter/Public/Characters/GSHeroCharacter.h/.cpp` |
| GameplayTags | `Config/DefaultGameplayTags.ini` |
| 全局 GAS 配置 | `Config/DefaultGame.ini` |
| AbilitySystemGlobals | `Source/GASShooter/Public/Characters/Abilities/GSAbilitySystemGlobals.h/.cpp` |
| GameplayCueManager | `Source/GASShooter/Public/Characters/Abilities/GSGameplayCueManager.h/.cpp` |
| 跳跃能力 | `Source/GASShooter/Public/Characters/Abilities/GSGA_CharacterJump.h/.cpp` |
| HUD | `Source/GASShooter/Public/UI/GSHUDWidget.h/.cpp` |
| 蓝图异步 (属性变化) | `Source/GASShooter/Public/Characters/Abilities/AsyncTaskAttributeChanged.h/.cpp` |
| 蓝图异步 (标签变化) | `Source/GASShooter/Public/Characters/Abilities/AsyncTaskGameplayTagAddedRemoved.h/.cpp` |
| PlayerController | `Source/GASShooter/Public/Player/GSPlayerController.h/.cpp` |
| GameMode | `Source/GASShooter/GASShooterGameModeBase.h/.cpp` |
| 角色移动组件 | `Source/GASShooter/Public/Characters/GSCharacterMovementComponent.h/.cpp` |
| 抛射物 | `Source/GASShooter/Public/Weapons/GSProjectile.h/.cpp` |
| 拾取物 | `Source/GASShooter/Public/Items/Pickups/GSPickup.h/.cpp` |
| 蓝图函数库 | `Source/GASShooter/Public/GSBlueprintFunctionLibrary.h/.cpp` |
| 引擎子系统 | `Source/GASShooter/Public/GSEngineSubsystem.h/.cpp` |
