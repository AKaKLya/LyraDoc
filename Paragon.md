# Paragon

## 工程



### GAS




---

### 游戏框架


`ASC` 依旧放在 `PlayerState` 里面，

由于分成了 `大厅 - 游戏` 这两部分内容，所以在大厅时 不赋予技能，到了游戏中 由GameMode来赋予技能，<br>
为了实现选择英雄的功能，每个玩家在选择英雄时，会把定义这个英雄行为的 `UNTXHeroInfo` 的 `FPrimaryAssetId` 存放在PlayerState中，

`UNTXHeroInfo` 定义了英雄所需要的技能、模型、动画蓝图，以及一些特殊的蒙太奇动画资产，<br>
游戏开始后，GameMode通过`FPrimaryAssetId`找到那个资产，传给`PlayerState::GiveDefaultAbility_Implementation`，由PS来完成后续部分.

技能在服务器上赋予，角色模型和动画使用RPC多播.

但是 大厅和游戏 不是同一个World，在切换World时会重新生成这些Actor，包括`PlayerState`， 所以为了跨关卡保留`UNTXHeroInfo`数据，<br>
`PlayerState`重写了`CopyProperties`，把这些值 设置给新的PS.

每个玩家选择好英雄之后，按下`Play`按钮 调用玩家控制器的`Server_NotifyReadyToPlay`，告诉大厅的GameMode选择好英雄了.<br>
当所有玩家都准备好以后，GameMode就切换到游戏地图了.

---

输入绑定

使用增强输入 模仿Lyra的做法， 有一点特殊的是 Ctrl键取消技能、左键确认技能 ， 这两个功能在蓝图里面直接使用AnyKey节点来写了，避免增强输入的优先级判定.<br>

有些技能需要玩家确认施放、取消施放， 所以就有了这个按键绑定.

确认、取消施放的原理是 玩家通过按键调用ASC的`LocalInputConfirm`，GA监听`LocalInputConfirm`的委托， 当玩家按下按键时， GA收到确认消息 执行后续流程.


技能的激活:<br>
当按下按键时，映射到对应的GameplayTag，
```cpp
void ANTXPlayerController::AbilityInputTagPressed(FGameplayTag InputTag)
void ANTXPlayerController::AbilityInputTagReleased(FGameplayTag InputTag)
```
这两个函数转发到ASC，ASC在Pressed中 收集与 `InputTag` 对应的技能 保存到TArray，Released中 把对应的技能从TArray移除.<br>

在这一帧的输入处理完毕后，
```cpp
void ANTXPlayerController::PostProcessInput(const float DeltaTime, const bool bGamePaused)
```
`PostProcessInput` 转发到ASC，ASC来处理收集到的技能，根据技能中的激活策略 激活技能.<br>
之后 清空TArray.


---


### Ability扩展

权限缓存

一个技能中可能会多次调用 `HasAuthority`、`IsLocallyControlled` 这样的函数，但是没有必要每次都重新获取，<br>
使用`TOptional`记录结果， 每次技能激活时 先缓存一下，<br>
这样一来 蓝图里面只需要调用`CachedAuthority`获取缓存结果.

```cpp
UCLASS()
class UNTXGameplayAbility : public UGameplayAbility
{
	TOptional<bool> bCachedAuthority;
	TOptional<bool> bCachedIsLocallyControlled;
}

void UNTXGameplayAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
	const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo,
	const FGameplayEventData* TriggerEventData)
{
	bCachedAuthority = HasAuthority(&ActivationInfo);
	bCachedIsLocallyControlled = IsLocallyControlled();
	Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}

UFUNCTION(BlueprintCallable,BlueprintPure = false,Category = "NTX|Ability",Meta = (ExpandBoolAsExecs = "ReturnValue"))
bool UNTXGameplayAbility::CachedAuthority()
{
	if (bCachedAuthority.IsSet())
	{
		return bCachedAuthority.GetValue();
	}

	bCachedAuthority = HasAuthority(&CurrentActivationInfo);
	return bCachedAuthority.GetValue();
}
```

---

攻击范围

有些技能需要显示一个Decal 告诉玩家伤害范围，射线检测等 检测手段也需要一个范围，<br>
GA里面定义一个攻击范围，供其他系统获取.

---

限定Tag

GA属性中有 `AbilityTags` 这样的标签，每次选择都要从一堆Tag里面找到Ability那个分组，<br>
但是又不能修改 `UGameplayAbility` 的源码，所以只能从反射信息上做手脚.

```cpp
UNTXGameplayAbility::UNTXGameplayAbility()
{
	InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;

#if WITH_EDITOR
	TMap<FName,FString> MetaData;
	MetaData.Add("categories","NTXAbility");
	for (FProperty* Property = UGameplayAbility::StaticClass()->PropertyLink; Property; Property = Property->PropertyLinkNext)
	{
		FString PropertyName = Property->GetName();
		if (PropertyName == "AbilityTags" || PropertyName == "CancelAbilitiesWithTag" || PropertyName == "BlockAbilitiesWithTag")
		{
			Property->AppendMetaData(MetaData);
		}
	}
#endif
}
```

这等效于 :

```cpp
UPROPERTY(/*...*/,meta = (categories = "NTXAbility"))
FGameplayTag AbilityTags;
```

---

## 英雄技能

### Gideon の 大招 - 扩展TargetData

RootMotionSouce

本来想通过获取范围内的敌人，给它们逐个应用 `RootMotionSouce` 实现黑洞吸附，<br>
但是 `AbilityTask_ApplyRootMotion` 只保存了角色自身的移动组件，<br>
如果不使用自带的，创建一个新的 `Task` 来实现这个功能也比较麻烦， 自带的 `Task`已经做好了网络相关的工作，<br>
所以 为什么不换一个视角，直接用自带的`Task` 做一个被动技能？


`UAbilityTask_ApplyRootMotion` 这些Task需要一堆参数，所以 把这些参数打包到 `FGameplayEventData`，使用 `SendGameplayEventToActor` 来触发技能.

但是，这里又会有一个问题， <br>
1.把参数塞进Event. <br>
2.服务器 --> 客户端 传输数据，这些参数需要通过网络传输过去，<br>

```cpp
static void SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload);
```

这个函数的Payload是 `FGameplayEventData`，它有一个多态成员 :
```cpp
/** The polymorphic target information for the event */
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = GameplayAbilityTriggerPayload)
FGameplayAbilityTargetDataHandle TargetData;

/**
*	FGameplayAbilityTargetDataHandle
*    主要用途如下：
*		- 避免在蓝图中复制整个目标数据结构
*		- 使目标数据结构支持多态
*		- 实现 NetSerialize，并在客户端/服务器之间按值复制
*
*		- 避免使用 UObject 虽然可以在蓝图中实现多态和按引用传递，但在复制时会遇到问题 :
*		- 按值复制
*		- 在蓝图中按引用传递
*		- TargetData 结构支持多态
*/
USTRUCT(BlueprintType)
struct GAMEPLAYABILITIES_API FGameplayAbilityTargetDataHandle
{
    /** Raw storage of target data, do not modify this directly */
	TArray<TSharedPtr<FGameplayAbilityTargetData>, TInlineAllocator<1> >	Data;
}
```

使用指针保存 `Data` 就有了修改其具体类型的机会，继承自 `FGameplayAbilityTargetData` 创建一个新的Data，就可以自定义数据内容了.

添加Data :
```cpp
template<typename T>
static void AddRootMotionTargetData(FGameplayAbilityTargetDataHandle& Handle, const T& NewTargetData, const bool bClearOld)
{
	static_assert(TIsDerivedFrom<T, FGameplayAbilityTargetData>::Value, "T must derive from FGameplayAbilityTargetData");
	if (bClearOld)
	{
		Handle.Clear();
	}
	T* Data = new T(NewTargetData);
	Handle.Add(Data);
}

void UNTXAbilitySystemStatics::AddRootMotionConstantTargetData(FGameplayAbilityTargetDataHandle& Handle,FNTXRootMotionConstantTargetData NewTargetData,bool bClearOld)
{
	AddRootMotionTargetData(Handle,NewTargetData,bClearOld);
}

void UNTXAbilitySystemStatics::AddRootMotionJumpTargetData(FGameplayAbilityTargetDataHandle& Handle,FNTXRootMotionJumpTargetData NewTargetData,bool bClearOld)
{
	AddRootMotionTargetData(Handle,NewTargetData,bClearOld);
}

void UNTXAbilitySystemStatics::AddRootMotionRadialTargetData(FGameplayAbilityTargetDataHandle& Handle,FNTXRootMotionRadialTargetData NewTargetData,bool bClearOld)
{
	AddRootMotionTargetData(Handle,NewTargetData,bClearOld);
}
```

获取Data :
```cpp
const FGameplayAbilityTargetData* TargetData = TriggerEventData->TargetData.Get(0);

const FNTXRootMotionRadialTargetData* Data = static_cast<const FNTXRootMotionRadialTargetData*>(TargetData);
const FNTXRootMotionJumpTargetData* Data = static_cast<const FNTXRootMotionJumpTargetData*>(TargetData);
const FNTXRootMotionConstantTargetData* Data = static_cast<const FNTXRootMotionConstantTargetData*>(TargetData);
```

或

```cpp
template<typename TargetDataType>
static const TargetDataType* GetTypedTargetDataFromHandle(const FGameplayAbilityTargetDataHandle& Handle, int32 Index)
{
	static_assert(TIsDerivedFrom<TargetDataType, FGameplayAbilityTargetData>::IsDerived,"TargetDataType must derive from FGameplayAbilityTargetData");

	const FGameplayAbilityTargetData* Data = Handle.Get(Index);
	if (Data && Data->GetScriptStruct() == TargetDataType::StaticStruct())
	{
		return static_cast<const TargetDataType*>(Data);
	}

	UE_LOG(LogTemp, Fatal, TEXT("GetTypedTargetData failed: invalid target data. Expected type: %s, Got: %s"),
	*TargetDataType::StaticStruct()->GetName(),Data ? *Data->GetScriptStruct()->GetName() : TEXT("nullptr"));
	
	return nullptr;
}
```

这样做的话 要保证传过来的一定是 static_cast 转换的类型，如何保证 ?

```cpp
UNTXGameplayAbility_RootMotionSouce::UNTXGameplayAbility_RootMotionSouce()
{
	FAbilityTriggerData TriggerData;
	TriggerData.TriggerTag = RootMotionSourceTag;
	AbilityTriggers.Add(TriggerData);
}

void UNTXGameplayAbility_RootMotionSouce::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
	const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo,
	const FGameplayEventData* TriggerEventData)
{
    const FGameplayAbilityTargetData* TargetData = TriggerEventData->TargetData.Get(0);
    if (TargetData == nullptr)
	{
        K2_EndAbility();
		return;
	}

    if (TriggerEventData->EventTag == ConstantForceTag)
	{
		ActiveTask = ApplyRootMotion<ConstantForce>(TargetData);
	}
	else if (TriggerEventData->EventTag == RadialForceTag)
	{
		ActiveTask = ApplyRootMotion<RadialForce>(TargetData);
	}
	else if (TriggerEventData->EventTag == JumpForceTag)
	{
		ActiveTask = ApplyRootMotion<JumpForce>(TargetData);
	}
}
```

GA只能由`RootMotionSourceTag`来触发，`SendGameplayEventToActor` 的 `EventTag` 指定为`RootMotionSourceTag` 的子Tag，<br>
之后 这个Event就会传到 `UNTXGameplayAbility_RootMotionSouce` ， 那么 `Data` 一定就是指定的类型.

还要定义网络复制所需的内容 :

```cpp
USTRUCT(BlueprintType)
struct NTX_API FNTXRootMotionRadialTargetData : public FGameplayAbilityTargetData
{
	GENERATED_BODY()

    virtual UScriptStruct* GetScriptStruct() const override { return FNTXRootMotionRadialTargetData::StaticStruct(); }
	bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess);
}

template<>
struct TStructOpsTypeTraits<FNTXRootMotionRadialTargetData> : public TStructOpsTypeTraitsBase2<FNTXRootMotionRadialTargetData>
{
	enum
	{
		WithNetSerializer = true,
	};
};
```

---

