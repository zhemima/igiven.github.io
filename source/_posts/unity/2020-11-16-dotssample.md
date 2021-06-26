

# ability

首先ability总是和item或者character结合在一起的



以下Authoring 会为ability加上不同的组件

- AbilityAutoRifleAuthoring  
- AbilityMovementAuthoring
- AbilitySprintAuthoring
- AbilityDeadAuthoring
- AbilitySelectSlotAuthoring
- AbilityMeleeAuthoring

```
EntityManager.CreateArchetype(
typeof(Ability.AbilityControl),  //控制器
typeof(Ability.AbilityTag), //技能tag( Melee,Movement,AutoRifle, SelectSlot,Dead,Sprint)
typeof(Ability.AbilityAction),  //技能动作(None,PrimaryFire,SecondaryFire,Reloading,Melee,NumActions)会通过Item关联具体值
typeof(settings),  // 技能的配置
typeof(state), // 技能状态  
typeof(PredictedState),  // 预测状态
typeof(InterpolatedState), // 插值状态
);
```



## AbilityCollection

`AbilityEntry`是一个IBufferElementData.存储了多个ability

定义了技能类型,能和什么类型一起运行,会打断什么类型

一个技能可能会有多个触发button,比如一把枪,会有开火键和装弹键





UpdateAbilityOwnership系统.. 填充character的OwnedCollection组件.分别为自己和item

再根据OwnedCollection填充ownedAbilities



PrepareOwnerForAbilityUpdate系统

检查ability是否启用.未启用则删除



SelectActiveBehavior系统

选择激活一个ability



# Item



```
EntityManager.CreateArchetype(
typeof(Item.InputState),  //输入状态
typeof(AbilityCollection.State),   //存储的是技能拥有者entity
typeof(AbilityCollection.AbilityEntry),//一个item可能有多个技能
);
```

Item.Initialize会查找所有拥有(Inventory.ItemEntry)的实体(就是character)

然后将item添加到character的Inventory.ItemEntry组件的buff中.

将item和character关联起来





Inventory.update系统

比较activeslot和lastactiveinventoryslot.来在item上添加和删除PartOwner.Visible组件





# Character



```
EntityManager.CreateArchetype(
typeof(Settings),  //角色设置
typeof(InterpolatedData), // 插值数据
typeof(PredictedData),//预测数据
typeof(ReplicatedData),// 角色类型

typeof(Player.OwnerPlayerId.Default),// 拥有者id
typeof(PlayerControlled.State),// 玩家控制器状态
typeof(HitColliderOwner.State),  //碰撞器状态

//Inventory
typeof(Inventory.ItemEntry)   //一个item的buff
typeof(Inventory.State)   //状态..激活的是哪一个item
typeof(Inventory.InternalState)  //最后激活的item

//生命和伤害
typeof(HealthStateData),
typeof(DamageEvent), //注意是个buff
typeof(DamageHistoryData),  

//ability
typeof(AbilityCollection.State),   //存储的是技能拥有者entity
typeof(AbilityCollection.AbilityEntry),//一个角色可能有多个能力
typeof(AbilityOwner.State),//技能拥有者状态
typeof(AbilityOwner.OwnedCollection),//技能拥有者是个buff  拥有者有可能是角色也有可能是item
typeof(AbilityOwner.OwnedAbility),//拥有的所有技能,包括角色和item的





//角色控制器数据
typeof(CharacterControllerComponentData),  // 一些角色控制器数据
typeof(CharacterControllerInitializationData),   //碰撞盒数据
typeof(CharacterControllerVelocity),
typeof(CharacterControllerMoveQuery),
typeof(CharacterControllerMoveResult),
typeof(CharacterControllerGroundSupportData),



);
```







# part

 Part 模块是抽象"部分关系"的一种存在, 不过具体到实现来说, 就是可以用来管理角色模型, 武器模型(以及其LOD), 以角色配置为例说明下述具体用法

```
EntityManager.CreateArchetype(
typeof(Part.Owner),  //部件拥有者
typeof(TerraformerWeaponPart.AutoringData)  //声音
typeof(TerraformerWeaponPart.AuthoringClass)  //材质,特效之类的
typeof(TerraformerWeaponPart.State)  //状态
);
```



同样的item和character都可以拥有part



# local player

```
EntityManager.CreateArchetype(
typeof(Player.State)  
typeof(UserCommand)    //是个buff
typeof(LocalPlayer)  
typeof(LocalPlayerCharacterControl.State)

typeof(AbilityUIElement)
typeof(PresentationElement)
typeof(PlayerCameraControl.State)
);




```









首先看`CharacterAuthoring`然后我们就知道.一个character到底有哪些东东了



//Assets/Unity.Sample.Terraformer/Prefabs/Characters/Terraformer/Char_Terraformer.prefab





SpawnCharacter









ItemAuthoring







# palyer 实例化流程

1. 客户端连接的时候,服务端PlayerModuleServer.CreatePlayerEntity 创建player实体
2. 服务端根据GameMode,创建角色GameModeSystemServer.Update()
3. 客户端创建localplayer.并根据系统将localplayer和character和player进行绑定