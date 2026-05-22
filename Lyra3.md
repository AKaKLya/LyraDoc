# Lyra网络



---


##### 开火技能

`ULyraGameplayAbility_RangedWeapon` 是 `Lyra` 中的射击技能。<br>
它充分利用了 GameplayAbility 预测系统，实现客户端预测射击（命中检测、弹药消耗、后坐力等），同时与服务器权威同步。

1.客户端预测：玩家开火时，客户端立即执行本地命中检测（PerformLocalTargeting），生成目标数据（命中位置、命中 Actor 等），并预测性地消耗弹药、增加武器散射、播放特效等。

2.发送到服务器：客户端将预测的目标数据通过 RPC 发送给服务器，同时附带一个预测键（PredictionKey）。

3.服务器验证：服务器收到目标数据后，重新进行命中检测（防止作弊），决定是否有效，并返回确认/拒绝。

4.客户端确认/回滚：服务器确认后，客户端的预测副作用被“承认”；如果拒绝，则回滚预测的效果（如恢复弹药、取消散射）。

---

开火时 绑定目标数据委托:
```cpp
void ULyraGameplayAbility_RangedWeapon::ActivateAbility(...)
{
    // 绑定目标数据就绪的回调
    OnTargetDataReadyCallbackDelegateHandle = MyAbilityComponent->AbilityTargetDataSetDelegate(
        CurrentSpecHandle, 
        CurrentActivationInfo.GetActivationPredictionKey()
    ).AddUObject(this, &ThisClass::OnTargetDataReadyCallback);
    
    // 更新武器上次开火时间
    WeaponData->UpdateFiringTime();
    
    Super::ActivateAbility(...);
}
```
`GetActivationPredictionKey()` 返回GA激活时生成的预测键（由 `TryActivateAbility` 产生）。<br>
该委托会在客户端本地生成目标数据或服务器收到目标数据时触发，是预测数据交换的桥梁。

---

开始目标定位（客户端预测）

```cpp
void ULyraGameplayAbility_RangedWeapon::StartRangedWeaponTargeting()
{
    // 创建预测作用域，使用GA激活时的预测键
    FScopedPredictionWindow ScopedPrediction(MyAbilityComponent, CurrentActivationInfo.GetActivationPredictionKey());

    // 本地执行命中检测（客户端预测）
    TArray<FHitResult> FoundHits;
    PerformLocalTargeting(FoundHits);
    
    // 构建目标数据
    FGameplayAbilityTargetDataHandle TargetData;
    // ... 填充命中结果
    
    // 立即处理目标数据（会触发 OnTargetDataReadyCallback）
    OnTargetDataReadyCallback(TargetData, FGameplayTag());
}
```

`FScopedPredictionWindow`：创建一个预测作用域。<br>
它会在客户端生成一个新的预测键（或复用现有键），并将该键设置为当前 AbilitySystemComponent 的 ScopedPredictionKey。<br>
作用域内产生的所有副作用（如 CommitAbility 消耗弹药）都会关联到这个键。

`PerformLocalTargeting` 内部根据武器配置（散射、射程等）执行多条射线检测，完全在客户端完成，实现零延迟的命中反馈。

实际上 这个函数里的 `FScopedPredictionWindow` 在后续版本中被移除了，只有下面的 `OnTargetDataReadyCallback` 函数中有预测键.

---

目标数据就绪回调（核心预测逻辑）
```cpp
void ULyraGameplayAbility_RangedWeapon::OnTargetDataReadyCallback(const FGameplayAbilityTargetDataHandle& InData, FGameplayTag ApplicationTag)
{
    // 再次创建预测作用域，确保后续 Commit 使用正确的预测键
    FScopedPredictionWindow ScopedPrediction(MyAbilityComponent);
    
    // 如果是本地控制的客户端且不是权威端，则发送目标数据到服务器
    if (CurrentActorInfo->IsLocallyControlled() && !CurrentActorInfo->IsNetAuthority())
    {
        MyAbilityComponent->CallServerSetReplicatedTargetData(
            CurrentSpecHandle,
            CurrentActivationInfo.GetActivationPredictionKey(),
            LocalTargetDataHandle,
            ApplicationTag,
            MyAbilityComponent->ScopedPredictionKey  // 传递预测键
        );
    }
    
    // 尝试消耗弹药等资源（CommitAbility）
    if (bIsTargetDataValid && CommitAbility(...))
    {
        // 增加散射（预测副作用）
        WeaponData->AddSpread();
        
        // 调用蓝图事件，应用伤害、触发 GameplayCue 等
        OnRangedWeaponTargetDataReady(LocalTargetDataHandle);
    }
    else
    {
        // 提交失败，立即结束能力（回滚）
        K2_EndAbility();
    }
    
    // 消耗已处理的目标数据（清理）
    MyAbilityComponent->ConsumeClientReplicatedTargetData(...);
}
```

FScopedPredictionWindow（无参构造）：<br>
创建一个新的预测窗口，生成新的预测键（或基于当前作用域）。<br>
这个键会与 CommitAbility 消耗的弹药、AddSpread 等副作用关联。

CallServerSetReplicatedTargetData：<br>
客户端将预测的目标数据和预测键发送给服务器。服务器会验证这些数据，并决定是否接受。服务器处理完后，会通过属性复制将结果返回给客户端。

CommitAbility：<br>
通常会在客户端预测性地消耗资源（如弹药），在服务器端真正消耗。<br>
由于它处于 FScopedPredictionWindow 内，如果服务器后来拒绝此预测，系统会自动回滚这些消耗。

ConsumeClientReplicatedTargetData：<br>
清除已处理的复制数据，并触发委托，让系统知道客户端已经收到了服务器的确认。

以下是对这一段的详细说明 ：

`FScopedPredictionWindow` 的工作原理 :<br>
RAII 类，它在构造和析构时修改 `UAbilitySystemComponent` 的内部状态。<br>

简化逻辑:<br>
创建 `FScopedPredictionWindow` 时，它会替换 ASC 的 ScopedPredictionKey 为一个新生成的键，并保存旧的键。<br>
窗口结束时，自动恢复之前的预测键.<br>
```cpp
// 简化自引擎源码
FScopedPredictionWindow::FScopedPredictionWindow(UAbilitySystemComponent* ASC, bool CanGenerateNewKey)
{
    // 保存旧的 ScopedPredictionKey
    OldKey = ASC->ScopedPredictionKey;
    
    if (CanGenerateNewKey)
    {
        // 生成一个新的预测键，并设置为 ASC 当前的 ScopedPredictionKey
        FPredictionKey NewKey = FPredictionKey::CreateNewPredictionKey(ASC);
        ASC->ScopedPredictionKey = NewKey;
    }
    // 也可以传入外部键，直接设置
}

FScopedPredictionWindow::~FScopedPredictionWindow()
{
    // 恢复旧的键
    ASC->ScopedPredictionKey = OldKey;
}
```

在这个窗口存活期间，`ASC->ScopedPredictionKey` 就是这个窗口对应的预测键。


`CommitAbility` 如何“自动”使用这个键？

```cpp
bool UGameplayAbility::CommitAbility(const FGameplayAbilitySpecHandle Handle,const FGameplayAbilityActorInfo* ActorInfo,
const FGameplayAbilityActivationInfo ActivationInfo,FGameplayTagContainer* OptionalRelevantTags)
{
    if (!CommitCheck(Handle, ActorInfo, ActivationInfo, OptionalRelevantTags))
        return false;
    CommitExecute(Handle, ActorInfo, ActivationInfo);
    K2_CommitExecute();
    ActorInfo->AbilitySystemComponent->NotifyAbilityCommit(this);
    return true;
}
```

其中 `CommitExecute` 调用 `ApplyCooldown` 和 `ApplyCost`，它们最终调用 `ApplyGameplayEffectToOwner`：

```cpp
void UGameplayAbility::ApplyCost(...) const
{
    UGameplayEffect* CostGE = GetCostGameplayEffect();
    if (CostGE)
    {
        ApplyGameplayEffectToOwner(Handle, ActorInfo, ActivationInfo, CostGE, GetAbilityLevel(Handle, ActorInfo));
    }
}
```

`ApplyGameplayEffectToOwner` 又会调用 `UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf`，并传递预测键:

```cpp
FActiveGameplayEffectHandle UGameplayAbility::ApplyGameplayEffectSpecToOwner(...) const
{
    FScopedGameplayCueSendContext GameplayCueSendContext;
    if (SpecHandle.IsValid() && (HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo)))
    {
        UAbilitySystemComponent* const AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();

        // 注意这一行, 调用 ASC 的 GetPredictionKeyForNewAction()
        return AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get(),
                    AbilitySystemComponent->GetPredictionKeyForNewAction());
    }
    return FActiveGameplayEffectHandle();
}
```

`GetPredictionKeyForNewAction()` 读取 `UAbilitySystemComponent::ScopedPredictionKey`。<br>
所以，当 `FScopedPredictionWindow` 存在时，`ASC->ScopedPredictionKey` 被设置为窗口的预测键，<br>
然后 `GetPredictionKeyForNewAction()` 返回该键，最终传给 `ApplyGameplayEffectSpecToSelf`。

因此，`CommitAbility` 与预测键的关联并非直接参数传递，而是通过 `ASC->ScopedPredictionKey` 这个局部的上下文变量实现的。

---

服务器如何通知客户端失败？

服务器处理客户端激活请求的函数是 `InternalServerTryActivateAbility`。<br>
如果服务器决定拒绝，它会调用：
```cpp
UFUNCTION(Client, Reliable)
void UAbilitySystemComponent::ClientActivateAbilityFailed_Implementation(FGameplayAbilitySpecHandle Handle, int16 PredictionKey)
{
    // 广播拒绝委托
    if (PredictionKey > 0)
    {
        FPredictionKeyDelegates::BroadcastRejectedDelegate(PredictionKey);
    }
    // 找到对应的 AbilitySpec 和 Ability 实例，结束它
    FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
    if (Spec)
    {
        // 标记激活被拒绝，并结束能力
        Spec->ActivationInfo.SetActivationRejected();
        TArray<UGameplayAbility*> Instances = Spec->GetAbilityInstances();
        for (UGameplayAbility* Ability : Instances)
        {
            if (Ability->CurrentActivationInfo.GetActivationPredictionKey().Current == PredictionKey)
            {
                Ability->CurrentActivationInfo.SetActivationRejected();
                Ability->K2_EndAbility();   // 结束能力，触发清理
            }
        }
    }
}
```
所以，服务器通过 RPC `ClientActivateAbilityFailed` 通知客户端，并传递预测键。

---

#### GE预测

`FPredictionKeyDelegates` 如何实现回滚？

`FPredictionKeyDelegates` 是一个全局委托管理器，维护一个 `TMap<PredictionKey, FDelegates>`。<br>
当客户端预测一个动作时（例如播放蒙太奇、应用 GameplayEffect），它会向该预测键注册一个“拒绝委托”，该委托用于撤销该动作。

在应用GE时，注册回滚操作:

```
 *	*** GameplayEffect 预测 ***
 *
 *	GameplayEffect 被视为预测的副作用，不会被显式询问。
 *	
 *	1. 只有在存在有效预测键时，GameplayEffect 才会在客户端上应用。（如果没有预测键，则在客户端上跳过应用）。
 *	2. 如果 GameplayEffect 是被预测的，那么属性、GameplayCue 和 GameplayTag 都会被预测。
 *	3. 当 FActiveGameplayEffect 被创建时，它会存储预测键（FActiveGameplayEffect::PredictionKey）
 *		3a. 即时效果在下面的“属性预测”中解释。
 *	4. 在服务器上，相同的预测键也会被设置在服务器将要复制下来的 FActiveGameplayEffect 上。
 *	5. 作为客户端，如果你收到一个带有有效预测键的复制 FActiveGameplayEffect，你会检查自己是否拥有相同键的 ActiveGameplayEffect，
 *     如果匹配，则我们不会执行“应用时”的逻辑，例如 GameplayCue。
 *     这解决了“重做”问题。但我们会在 ActiveGameplayEffects 容器中暂时拥有两个“相同”的 GameplayEffect：
 *	6. 同时，UAbilitySystemComponent::ReplicatedPredictionKey 会跟上，预测性的效果会被移除。当它们在这种情况下被移除时，我们再次检查 PredictionKey，
 *		并决定是否应该执行“移除时”逻辑 / GameplayCue。
 *		
 *	至此，我们有效地将 GameplayEffect 作为副作用进行了预测，并处理了“撤销”和“重做”问题。
```

```cpp
FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(const FGameplayEffectSpec& Spec, FPredictionKey& InPredictionKey, bool& bFoundExistingStackableGE)
{
    // Once replicated state has caught up to this prediction key, we must remove this gameplay effect.
	InPredictionKey.NewRejectOrCaughtUpDelegate(FPredictionKeyEvent::CreateUObject(Owner, &UAbilitySystemComponent::RemoveActiveGameplayEffect_NoReturn, AppliedActiveGE->Handle, -1));
}
```

```cpp
void FPredictionKey::NewRejectOrCaughtUpDelegate(FPredictionKeyEvent Event)
{
	FPredictionKeyDelegates::NewRejectOrCaughtUpDelegate(Current, Event);
}

static void FPredictionKeyDelegates::NewRejectOrCaughtUpDelegate(FPredictionKey::KeyType Key, FPredictionKeyEvent NewEvent)
{
	FDelegates& Delegates = Get().DelegateMap.FindOrAdd(Key);
	Delegates.CaughtUpDelegates.Add(NewEvent);
	Delegates.RejectedDelegates.Add(NewEvent);
}
```

通过 `FPredictionKey` 的ID 向全局委托注册一个新的委托，当捕捉到对应的ID时，执行对应的函数.<br>
这里是 捕捉到对应的ID时，移除这个GE.

---

能力结束时的清理
```cpp
void ULyraGameplayAbility_RangedWeapon::EndAbility(...)
{
    // 移除目标数据委托
    MyAbilityComponent->AbilityTargetDataSetDelegate(...).Remove(...);
    // 消耗残留的目标数据
    MyAbilityComponent->ConsumeClientReplicatedTargetData(...);
    
    Super::EndAbility(...);
}
```

---


```cpp
客户端                                    服务器
  |                                         |
  | 1. ActivateAbility                      |
  |    绑定目标数据委托                       |
  |                                         |
  | 2. StartRangedWeaponTargeting           |
  |    - 创建 FScopedPredictionWindow       |
  |    - 本地命中检测                        |
  |    - 构建 TargetData                    |
  |    - 调用 OnTargetDataReadyCallback     |
  |                                         |
  | 3. OnTargetDataReadyCallback            |
  |    - 创建新预测窗口                       |
  |    - CallServerSetReplicatedTargetData ---> 4. ServerSetReplicatedTargetData
  |    - CommitAbility (消耗弹药, 增加散射)   |     - 创建预测窗口
  |    - OnRangedWeaponTargetDataReady       |     - 存储 TargetData
  |      (应用伤害/特效)                      |     - 广播委托，触发服务器端 OnTargetDataReadyCallback
  |                                         |
  |                                         | 5. 服务器端 OnTargetDataReadyCallback
  |                                         |    - CommitAbility (消耗服务器资源)
  |                                         |    - 应用伤害
  |                                         |    - 发送确认/替换信息给客户端
  |                                         |
  | 6. 收到服务器确认                        |
  |    - 更新命中标记                        |
  |    - 预测键被确认，预测的 Effect 被移除   |
  |    - 最终状态与服务器同步                 |
  |                                         |
  | 7. EndAbility (清理委托和缓存)           |
```

技能激活 :
```cpp
void ULyraGameplayAbility_RangedWeapon::ActivateAbility(...)
{
    // 1. 绑定目标数据就绪委托
    UAbilitySystemComponent* MyAbilityComponent = CurrentActorInfo->AbilitySystemComponent.Get();

    OnTargetDataReadyCallbackDelegateHandle = MyAbilityComponent->AbilityTargetDataSetDelegate(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey())
    .AddUObject(this, &ThisClass::OnTargetDataReadyCallback);

    // 2. 更新武器实例的最后开火时间（用于射速限制等）
    ULyraRangedWeaponInstance* WeaponData = GetWeaponInstance();
    WeaponData->UpdateFiringTime();

    // 3. 调用父类 ActivateAbility（最终会调用蓝图事件等）
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}
```

`AbilityTargetDataSetDelegate` 是一个委托，当客户端本地生成目标数据或服务器收到目标数据时触发。这里绑定到 `OnTargetDataReadyCallback`。<br>
传入的 `CurrentActivationInfo.GetActivationPredictionKey()` 是能力激活时生成的预测键（由 `InternalTryActivateAbility` 设置）。<br>
父类 `ActivateAbility`  调用蓝图事件 `K2_ActivateAbility`。<br>

---

目标定位：StartRangedWeaponTargeting <br>
由蓝图调用

```cpp
void ULyraGameplayAbility_RangedWeapon::StartRangedWeaponTargeting()
{
    // 1. 获取必要组件
    UAbilitySystemComponent* MyAbilityComponent = CurrentActorInfo->AbilitySystemComponent.Get();
    AController* Controller = GetControllerFromActorInfo();
    ULyraWeaponStateComponent* WeaponStateComponent = Controller->FindComponentByClass<ULyraWeaponStateComponent>();

    // 2. 创建预测作用域，使用能力激活时的预测键
    FScopedPredictionWindow ScopedPrediction(MyAbilityComponent, CurrentActivationInfo.GetActivationPredictionKey());

    // 3. 执行本地命中检测（客户端预测）
    TArray<FHitResult> FoundHits;
    PerformLocalTargeting(FoundHits);

    // 4. 构建目标数据
    FGameplayAbilityTargetDataHandle TargetData;
    TargetData.UniqueId = WeaponStateComponent ? WeaponStateComponent->GetUnconfirmedServerSideHitMarkerCount() : 0;
    const int32 CartridgeID = FMath::Rand();
    for (const FHitResult& FoundHit : FoundHits) 
    {
        FLyraGameplayAbilityTargetData_SingleTargetHit* NewTargetData = new FLyraGameplayAbilityTargetData_SingleTargetHit();
        NewTargetData->HitResult = FoundHit;
        NewTargetData->CartridgeID = CartridgeID;
        TargetData.Add(NewTargetData);
    }

    // 5. 记录未确认的命中标记（用于后续服务器确认）
    if (WeaponStateComponent) 
    {
        WeaponStateComponent->AddUnconfirmedServerSideHitMarkers(TargetData, FoundHits);
    }

    // 6. 立即调用回调，处理目标数据
    OnTargetDataReadyCallback(TargetData, FGameplayTag());
}
```

`FScopedPredictionWindow`：构造时传入能力激活的预测键，使得该作用域内的所有 `ASC->ScopedPredictionKey` 都指向这个键。<br>
后续调用 `CommitAbility` 或应用 `GameplayEffect` 都会自动关联该键。

`PerformLocalTargeting`：内部根据武器散射、射程等参数，执行射线检测，返回命中结果。<br>
由客户端本地计算，零延迟。

`AddUnconfirmedServerSideHitMarkers`：将本次预测的命中结果缓存到 `ULyraWeaponStateComponent`，用于后续服务器确认时匹配。

调用 `OnTargetDataReadyCallback`：确保了目标数据在本地被“消费”，触发后续的提交GA、增加散射、发送 RPC 等。

---

目标数据就绪回调：OnTargetDataReadyCallback <br>
这个函数既在客户端本地调用（如上），也会在服务器通过 RPC 调用时被触发。它负责提交GA、发送 RPC、以及执行蓝图逻辑。

```cpp
void ULyraGameplayAbility_RangedWeapon::OnTargetDataReadyCallback(const FGameplayAbilityTargetDataHandle& InData, FGameplayTag ApplicationTag)
{
    UAbilitySystemComponent* MyAbilityComponent = CurrentActorInfo->AbilitySystemComponent.Get();

    // 1. 创建预测作用域（无参数表示生成新键或复用当前）
    FScopedPredictionWindow ScopedPrediction(MyAbilityComponent);

    // 2. 获取目标数据的本地副本（避免外部修改）
    FGameplayAbilityTargetDataHandle LocalTargetDataHandle(MoveTemp(const_cast<FGameplayAbilityTargetDataHandle&>(InData)));

    // 3. 如果是本地控制的客户端且不是权威端，将目标数据发送给服务器
    const bool bShouldNotifyServer = CurrentActorInfo->IsLocallyControlled() && !CurrentActorInfo->IsNetAuthority();
    if (bShouldNotifyServer) 
    {
        MyAbilityComponent->CallServerSetReplicatedTargetData(CurrentSpecHandle,CurrentActivationInfo.GetActivationPredictionKey(),
        LocalTargetDataHandle,ApplicationTag,MyAbilityComponent->ScopedPredictionKey);
    }

    // 4. 提交能力（消耗弹药、检查冷却等）
    const bool bIsTargetDataValid = true; // 本例中始终为真，实际可验证
    if (bIsTargetDataValid && CommitAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo)) 
    {
        // 增加武器散射（预测副作用）
        ULyraRangedWeaponInstance* WeaponData = GetWeaponInstance();
        WeaponData->AddSpread();

        // 调用蓝图事件，应用伤害、触发 GameplayCue 等
        OnRangedWeaponTargetDataReady(LocalTargetDataHandle);
    } 
    else 
    {
        // 提交失败，立即结束能力
        K2_EndAbility();
    }

    // 5. 消费已处理的目标数据（清理缓存）
    MyAbilityComponent->ConsumeClientReplicatedTargetData(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey());
}
```


```cpp
const bool bShouldNotifyServer = CurrentActorInfo->IsLocallyControlled() && !CurrentActorInfo->IsNetAuthority();
```
`IsLocallyControlled()`：<br>
返回 `true` 表示该 `Ability` 的 `Avatar Actor` 由本地玩家控制 - 即本地客户端上的自主代理 `ROLE_AutonomousProxy`，<br>
或者由本地 AI 控制器控制（单机或监听服务器）。<br>
`IsNetAuthority()`：返回 `true` 表示当前运行在服务器（权威端）上。<br>

以上这两种情况不需要向服务器发送数据, 仅在客户端（非服务器）且该角色由本地控制时，才需要将目标数据发送给服务器.




`CallServerSetReplicatedTargetData`：客户端将命中数据和预测键发送给服务器。<br>
服务器会重新验证命中，并返回确认或替换结果。

`CommitAbility`：调用 `UGameplayAbility::CommitAbility`，内部会应用消耗弹药的 `GameplayEffect`。<br>
由于处于 `FScopedPredictionWindow` 作用域内，该 `Effect` 会关联到窗口的预测键，从而可被回滚。

`OnRangedWeaponTargetDataReady`：蓝图实现的事件，通常在此处应用伤害、触发 GameplayCue、播放特效等。<br>
这些操作如果使用 `ApplyGameplayEffectToTarget` 等，也会自动使用当前预测键。

`ConsumeClientReplicatedTargetData`：清除已处理的复制数据，并通知系统该预测键的目标数据已被消费。

---

服务器端处理（RPC）

当客户端调用 `CallServerSetReplicatedTargetData` 时，<br>
最终会执行服务器端的 `ServerSetReplicatedTargetData_Implementation`: <br>
1.创建 `FScopedPredictionWindow`，使用客户端传来的 `CurrentPredictionKey`。<br>
2.将目标数据存入 `AbilityTargetDataMap`，并触发 `TargetSetDelegate` 广播。<br>
3.这个广播会调用客户端已绑定的 `OnTargetDataReadyCallback`（在服务器上执行）。<br>


服务器上的 `OnTargetDataReadyCallback` 会执行以下逻辑（在 `WITH_SERVER_CODE` 块内）：<br>
如果服务器是权威端，会通过 `ULyraWeaponStateComponent::ClientConfirmTargetData` 向客户端发送确认，包括哪些命中被替换。<br>
同样会调用 `CommitAbility`（消耗服务器端的资源），并执行 `OnRangedWeaponTargetDataReady` 来应用伤害等。<br>
如果客户端预测的命中结果与服务器不一致，服务器会发送替换信息，客户端会更新命中标记。<br>

---

预测键的确认与回滚 :

客户端提交失败时回滚:<br>
如果 `CommitAbility` 返回 false（例如弹药不足），客户端会调用 K2_EndAbility。此时能力结束，<br>
`CommitAbility` 失败，根本没有创建 Effect，所以无需额外回滚。

服务器拒绝激活:<br>
如果服务器认为此次开火无效（例如客户端作弊、射程外等），它会调用 `ClientActivateAbilityFailed` RPC，携带预测键。<br>
客户端收到后：<br>
1.`FPredictionKeyDelegates::BroadcastRejectedDelegate(PredictionKey)` 触发所有注册的拒绝委托。<br>
2.其中最重要的委托是 `ApplyGameplayEffectSpec` 中注册的 `RemoveActiveGameplayEffect_NoReturn`，它会移除所有关联该预测键的 GE（弹药消耗、散射增加等）。<br>
3.同时能力实例被标记为拒绝并结束，所有预测的副作用被撤销。<br>

服务器接受并确认:<br>
服务器成功处理后，会通过属性复制将 `ReplicatedPredictionKey` 设置为该预测键。客户端收到后，会调用 `OnRep_PredictionKey`，触发“CaughtUp”委托。<br>
此时：<br>
客户端预测的 `GameplayEffect` 会被移除，但服务器复制的相同 `Effect` 已经存在，因此最终效果保持不变。<br>
散射增加等副作用如果通过属性复制同步，也会被服务器的值覆盖。<br>

---

```cpp
客户端                              服务器
  |                                   |
  | ActivateAbility 绑定委托          |
  |                                   |
  | StartRangedWeaponTargeting        |
  | - 本地命中检测                     |
  | - 调用 OnTargetDataReadyCallback  |
  |   (本地触发委托)                   |
  |   - 发送 RPC -------------------->|
  |   - 本地 CommitAbility            |
  |   - 本地应用特效/伤害              |
  |                                   | 收到 RPC: ServerSetReplicatedTargetData
  |                                   | - 存储 TargetData
  |                                   | - 广播委托 -> 服务器端 OnTargetDataReadyCallback
  |                                   |   - 服务器 CommitAbility
  |                                   |   - 服务器应用伤害
  |                                   |   - 调用 ClientConfirmTargetData --->
  |                                   |
  | 收到 ClientConfirmTargetData      |
  | - 更新命中标记                     |
  |                                   |
  | 属性复制追上预测键 -> 清理预测 Effect |
```

可能发生回滚的情况:

1. 能力激活被服务器拒绝 <br>
位置：服务器端 `CanActivateAbility` 返回 false。<br>
服务器动作：调用 `ClientActivateAbilityFailed(Handle, PredictionKey.Current)`。<br>
客户端回滚：`ClientActivateAbilityFailed_Implementation` 广播 `BroadcastRejectedDelegate`，<br>
移除所有带有该预测键的 GE（包括尚未发生的任何副作用）。<br>
由于此时 CommitAbility 尚未执行，通常没有副作用，但激活本身被标记为拒绝。

2. 服务器端 CommitAbility 失败 <br>
位置：服务器端 `OnTargetDataReadyCallback` 中调用 `CommitAbility` 返回 false。

服务器动作：<br>
服务器不会调用 `ClientActivateAbilityFailed`，而是直接 `K2_EndAbility` 结束能力。<br>
但是，客户端已经通过本地的 `CommitAbility` 消耗了弹药并增加了散射（这些是预测副作用）。<br>
服务器会通过属性同步 强制修改客户端的数据 改为正确值.




---


 ```
 *	*** 属性预测 ***
 *	
 *	由于属性作为标准 UProperty 进行复制，预测对它们的修改可能会很棘手（“覆盖”问题）。瞬时修改甚至更难，因为它们在本质上是无状态的。
 *	（例如，如果没有修改之外的记账，回滚属性修改是很困难的）。这使得“撤销”和“重做”问题在这种情况下也很困难。
 *	
 *	基本的策略是将属性预测视为增量预测而不是绝对值预测。我们不预测我们有 90 点法力值，而是预测我们相比服务器值有 -10 点法力值，直到服务器确认我们的预测键。
 *	基本上，在预测性地进行瞬时修改时，将其视为对属性的 /无限持续时间的修改/。这解决了“撤销”和“重做”问题。
 *	
 *	对于“覆盖”问题，我们可以在属性的 OnRep 中通过将复制的（服务器）值视为属性的“基础值”而不是“最终值”来处理，并在复制发生后重新聚合我们的“最终值”。
 *	
 *	
 *	1. 我们将预测性的瞬时 GameplayEffect 视为无限持续时间的 GameplayEffect。参见 UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf。
 *	2. 我们必须 *总是* 接收属性的 RepNotify 调用（而不仅仅是在上次本地值发生变化时，因为我们会提前预测变化）。使用 REPNOTIFY_Always 实现。
 *	3. 在属性 RepNotify 中，我们调用 AbilitySystemComponent::ActiveGameplayEffects，根据新的“基础值”更新我们的“最终值”。GAMEPLAYATTRIBUTE_REPNOTIFY 可以做到这一点。
 *	4. 其他一切都像上面一样工作（GameplayEffect 预测）：当预测键被跟上时，预测性的 GameplayEffect 被移除，我们将恢复到服务器给定的值。
 *	
 *	
 *	示例：
 *	
 *	void UMyHealthSet::GetLifetimeReplicatedProps(TArray< FLifetimeProperty > & OutLifetimeProps) const
 *	{
 *		Super::GetLifetimeReplicatedProps(OutLifetimeProps);
 *
 *		DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, Health, COND_None, REPNOTIFY_Always);
 *	}
 *	
 *  void UMyHealthSet::OnRep_Health()
 *  {
 *		GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, Health);
 *  }
 ```

---

 #### Cue
 ```
 *  
 *  *** GameplayCue 事件 ***
 *  
 *  除了已经解释过的 GameplayEffect 之外，GameplayCue 也可以独立激活。这些函数（UAbilitySystemComponent::ExecuteGameplayCue 等）会考虑网络角色和预测键。
 *  
 *  1. 在 UAbilitySystemComponent::ExecuteGameplayCue 中，如果是权威端，则执行多播事件（带有复制键）。如果是非权威端但有有效的预测键，则预测 GameplayCue。
 *  2. 在接收端（NetMulticast_InvokeGameplayCueExecuted 等），如果存在复制键，则不执行事件（假设你已经预测了它）。
 *  
 *  记住，FPredictionKey 只会复制给发起方所有者。这是 FReplicationKey 的内在属性。
 *	
 *	*** 触发数据预测 ***
 *	
 *	触发数据当前用于激活能力。本质上，这都经过与 ActivateAbility 相同的代码路径。能力不是由输入按键激活，而是由其他游戏代码驱动的事件激活。
 *	客户端能够预测性地执行这些事件，从而预测性地激活能力。
 *	
 *	然而，这里有一些细微差别，因为服务器也会运行触发事件的代码。服务器不会只是等待客户端的消息。服务器会维护一个列表，记录哪些触发的能力已经从预测性能力中被激活。
 *	当从触发能力收到 TryActivate 时，服务器会查看 /它自己/ 是否已经运行了这个能力，并用该信息进行响应。
 *	
 *	触发事件和复制方面还有工作要做（在结尾处解释）。
 *	
 *	---------------------------------------------------------	
 ```

#### 高级话题
 ```	
 *	高级话题！	
 *	
 *	*** 依赖关系 ***
 *	
 *	我们可能会遇到这样的情况：“能力 X 激活并立即触发一个事件，该事件激活能力 Y，后者再激活能力 Z”。依赖链是 X->Y->Z。
 *	这些能力中的每一个都可能被服务器拒绝。如果 Y 被拒绝，那么 Z 也永远不会发生，但服务器从未尝试运行 Z，所以服务器没有明确决定“Z 不能运行”。
 *	
 *	为了处理这种情况，我们有一个 Base PredictionKey 的概念，它是 FPredictionKey 的一个成员。当调用 TryActivateAbility 时，我们会传入当前的 PredictionKey（如果适用）。
 *	该预测键被用作任何新生成的预测键的 Base。我们通过这种方式构建一个键链，并且如果 Y 被拒绝，我们可以使 Z 无效。
 *	
 *	不过这稍微有些微妙。在 X->Y->Z 的情况下，服务器在尝试运行链本身之前只会收到 X 的 PredictionKey。例如，它会使用客户端发送给它的原始预测键来 TryActivate Y 和 Z，
 *	而客户端每次调用 TryActivateAbility 时都会生成一个新的 PredictionKey。客户端 *必须* 为每次能力激活生成一个新的 PredictionKey，因为每次激活在逻辑上不是原子性的。
 *	在事件链中产生的每个副作用都必须有一个唯一的 PredictionKey。我们不能让 X 中产生的 GameplayEffect 与 Z 中产生的具有相同的 PredictionKey。
 *	
 *	为了解决这个问题，X 的预测键被视为 Y 和 Z 的 Base 键。从 Y 到 Z 的依赖关系完全保持在客户端侧，通过 FPredictionKeyDelegates::AddDependency 实现。
 *	如果 Y 被拒绝/确认，我们会添加委托来拒绝/跟上 Z。
 *	
 *	这个依赖系统允许我们在一个预测窗口/作用域内拥有多个在逻辑上不是原子性的预测性动作。
 *
 *	
 *	
 *	*** 能力内的额外预测窗口 ***
 *	
 *	如前所述，一个预测键仅在一个逻辑作用域内有效。一旦 ActivateAbility 返回，我们基本上就完成了该键的使用。如果能力正在等待外部事件或计时器，到它返回时，
 *	我们可能已经收到了服务器的确认/拒绝。此后产生的任何副作用将不再与原始键的生命周期绑定。
 *	
 *	这并不太糟糕，除了能力有时会希望响应用户输入。例如，“按住并蓄力”的能力希望在按钮释放时立即预测一些内容。可以使用 FScopedPredictionWindow 在能力内创建一个新的预测窗口。
 *	
 *	FScopedPredictionWindow 提供了一种方式，向服务器发送一个新的预测键，并让服务器在同一个逻辑作用域内接收和使用该键。
 *	
 *	UAbilityTask_WaitInputRelease::OnReleaseCallback 是一个很好的例子。事件流程如下：
 *	1. 客户端进入 UAbilityTask_WaitInputRelease::OnReleaseCallback 并启动一个新的 FScopedPredictionWindow。这会为此作用域创建一个新的预测键（FScopedPredictionWindow::ScopedPredictionKey）。
 *	2. 客户端调用 AbilitySystemComponent->ServerInputRelease，并将 ScopedPrediction.ScopedPredictionKey 作为参数传递。
 *	3. 服务器运行 ServerInputRelease_Implementation，它接收传入的 PredictionKey，并通过 FScopedPredictionWindow 将其设置为 UAbilitySystemComponent::ScopedPredictionKey。
 *	4. 服务器 /在同一作用域内/ 运行 UAbilityTask_WaitInputRelease::OnReleaseCallback。
 *	5. 当服务器在 ::OnReleaseCallback 中遇到 FScopedPredictionWindow 时，它会从 UAbilitySystemComponent::ScopedPredictionKey 获取预测键。然后该键将用于此逻辑作用域内的所有副作用。
 *	6. 一旦服务器结束此作用域预测窗口，所使用的预测键就会被完成并设置为 ReplicatedPredictionKey。
 *	7. 在此作用域中创建的所有副作用现在都在客户端和服务器之间共享一个键。
 *	
 *	这能够工作的关键是 ::OnReleaseCallback 调用 ::ServerInputRelease，后者又在服务器上调用 ::OnReleaseCallback。没有其他东西可以使用该给定的预测键。
 *	
 *	虽然此示例中没有“Try/Failed/Succeed”调用，但所有副作用在过程上是分组的/原子性的。这解决了任意在服务器和客户端上运行的函数调用的“撤销”和“重做”问题。
 *	
 *	
 *	---------------------------------------------------------
 *	
 *	不支持 / 问题 / 待办
 *	
 *	触发事件不会显式复制。例如，如果一个触发事件只在服务器上运行，客户端永远不会知道。这也阻止了我们进行跨玩家/AI 等事件。最终应该添加对此的支持，
 *	并且应该遵循 GameplayEffect 和 GameplayCue 相同的模式（使用预测键预测触发事件，如果 RPC 事件带有预测键则忽略它）。
 *	
 *		 
 *	*** 预测“元”属性（如伤害/治疗）与“真实”属性（如生命值） ***
 *	
 *	我们无法预测性地应用元属性。元属性仅作用于即时效果，在 GameplayEffect 的后端（UAttributeSet 的 Pre/Post Modify Attribute）中起作用。当应用基于持续时间的 GameplayEffect 时，
 *	这些事件不会被调用。例如，一个持续 5 秒修改伤害的 GameplayEffect 是没有意义的。
 *	
 *	为了支持这一点，我们可能会添加对基于持续时间的元属性的有限支持，并将即时 GameplayEffect 的转换从前端（UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf）
 *	移到后端（UAttributeSet::PostModifyAttribute）。
 * 
 *			 
 *	*** 预测正在进行的乘法性 GameplayEffect ***
 *	
 *	在预测基于百分比的 GameplayEffect 时也存在限制。由于服务器只复制属性的“最终值”，而不是整个修改它的聚合器链，我们可能会遇到客户端无法准确预测新 GameplayEffect 的情况。
 *	
 *	例如： 
 *	- 客户端有一个永久的 +10% 移动速度增益，基础移动速度为 500 -> 该客户端的最终移动速度为 550。
 *	- 客户端有一个能力，提供额外的 10% 移动速度增益。预期是将基于百分比的乘数 *求和* 以获得 20% 的总增益，使 500 -> 600 移动速度。
 *	- 然而在客户端，我们只是对 550 应用 10% 的增益 -> 605。
 *	
 *	这需要通过复制属性的聚合器链来解决。我们已经复制了其中的一些数据，但不是完整的修饰符列表。我们最终需要研究支持这一点。
 *	 
 *	
 *	*** “弱预测” ***
 *	
 *	可能仍然存在不完全适合此系统的情况。某些情况下，预测键交换是不可行的。例如，一个能力使得任何与该玩家碰撞/接触的玩家都会收到一个减速的 GameplayEffect 并改变其材质颜色。
 *	由于我们不能每次发生这种情况都发送服务器 RPC（而且服务器在其模拟的时间点不一定能处理该消息），因此没有办法关联客户端和服务器之间的 GameplayEffect 副作用。
 *	
 *	这里的一种方法可能是考虑一种较弱形式的预测。在这种形式中，不使用新的预测键，而是服务器假设客户端将预测一个完整能力的所有副作用。这至少可以解决“重做”问题，
 *	但不能解决“完整性”问题。如果客户端侧的预测可以做得尽可能小——例如只预测初始粒子效果，而不预测状态和属性变化——那么问题就会不那么严重。
 *	
 *	我可以设想一种弱预测模式，当没有新的预测键可以准确关联副作用时，（某些能力？所有能力？）会回退到这种模式。在弱预测模式下，也许只能预测某些动作，
 *	例如 GameplayCue 执行事件，而不是 OnAdded/OnRemove 事件。
 *	
 *	
 */
```


---
