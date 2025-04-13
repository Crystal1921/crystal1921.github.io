---
title: Brain系统笔记
date: 2025-4-13 20:57:00
tags: 
  - java
  - mod
  - mc
---

mc中复杂的实体逻辑通过`Brain`系统实现，为了使用这套系统，我们需要在实体类中覆写以下方法来为这个实体添加一个大脑

``` java
    @Override
    @SuppressWarnings("all")
    public Brain<MarisaEntity> getBrain() {
        return (Brain<MarisaEntity>) super.getBrain();
    }

    @Override
    protected Brain.@NotNull Provider<MarisaEntity> brainProvider() {
        return Brain.provider(MarisaBrain.getMemoryTypes(), MarisaBrain.getSensorTypes());
    }

    @Override
    protected @NotNull Brain<?> makeBrain(@NotNull Dynamic<?> dynamicIn) {
        Brain<MarisaEntity> brain = this.brainProvider().makeBrain(dynamicIn);
        MarisaBrain.registerBrainGoals(brain, this);
        return brain;
    }
```

然后我们需要使得`brain`接管该实体的动作，并实时更新`brain`的状态

``` java
    @Override
    protected void customServerAiStep() {
        this.getBrain().tick((ServerLevel) this.level(), this);
        MarisaBrain.updateActivity(this);
        super.customServerAiStep();
    }
```

下面我们来到了最重要的一部分：如何构建这个复杂的大脑。我们可以先看看`brainProvider`中这两个方法`MarisaBrain.getMemoryTypes()`和`MarisaBrain.getSensorTypes()`

``` java
    public static ImmutableList<MemoryModuleType<?>> getMemoryTypes() {
        List<MemoryModuleType<?>> defaultTypes = Lists.newArrayList(
                MemoryModuleType.PATH,
                MemoryModuleType.LOOK_TARGET,
                MemoryModuleType.NEAREST_HOSTILE,
                MemoryModuleType.HURT_BY,
                MemoryModuleType.HURT_BY_ENTITY,
                MemoryModuleType.CANT_REACH_WALK_TARGET_SINCE,
                MemoryModuleType.WALK_TARGET,
                MemoryModuleType.ATTACK_TARGET,
                MemoryModuleType.ATTACK_COOLING_DOWN,
                MemoryModuleType.NEAREST_VISIBLE_LIVING_ENTITIES,
                MemoryModuleType.ANGRY_AT,
                MemoryModuleType.NEAREST_VISIBLE_ATTACKABLE_PLAYER
        );
        return ImmutableList.copyOf(defaultTypes);
    }
```

在这个方法中，我们返回了一个不可变列表，这里记录了该实体所需要的记忆种类，这样我们就可以把对应种类的数据存在Brain中，以供调用。

``` java
    public static ImmutableList<SensorType<? extends Sensor<? super MarisaEntity>>> getSensorTypes() {
        List<SensorType<? extends Sensor<? super MarisaEntity>>> defaultTypes = Lists.newArrayList(
                SensorType.NEAREST_PLAYERS,
                SensorType.HURT_BY
        );
        return ImmutableList.copyOf(defaultTypes);
    }
```

在这个方法中，我们依然返回了一个不可变列表，它是该实体会使用的监听器，这些监听器每tick执行，来实行操作。比如在`SensorType.NEAREST_PLAYERS`的注册中，我们看到是一个`PlayerSensor`实例，在其中会把最近的玩家写入`MemoryModuleType.NEAREST_VISIBLE_ATTACKABLE_PLAYER`

在`Brain.tick()`方法中，我们看到了实际上大脑的逻辑中，监听器会先于实体任务执行。

``` java
    public void tick(ServerLevel level, E entity) {
        this.forgetOutdatedMemories();
        this.tickSensors(level, entity);
        this.startEachNonRunningBehavior(level, entity);
        this.tickEachRunningBehavior(level, entity);
    }

        private void tickSensors(ServerLevel level, E brainHolder) {
        for (Sensor<? super E> sensor : this.sensors.values()) {
            sensor.tick(level, brainHolder);
        }
    }
```

下面一部分是大脑中最重要的部分：实体执行的任务。

``` java
    public static void registerBrainGoals(Brain<MarisaEntity> brain, MarisaEntity marisa) {
        registerCoreGoals(brain);
        registerIdleGoals(brain);
        registerAggressiveGoals(brain, marisa);
        registerMagicReleaseGoals(brain, marisa);

        brain.setCoreActivities(ImmutableSet.of(Activity.CORE));
        brain.setDefaultActivity(Activity.IDLE);
        brain.useDefaultActivity();
        updateActivity(marisa, brain);
    }

        public static void updateActivity(MarisaEntity maid, Brain<MarisaEntity> brain) {
        if (maid.level() instanceof ServerLevel serverLevel) {
            long gameTime = serverLevel.getGameTime();
            long dayTime = serverLevel.getDayTime();
            brain.updateActivityFromSchedule(dayTime, serverLevel.getGameTime());
        }
    }
```

这里我们分别注册了核心目标，默认目标，攻击目标，魔法施放目标。

核心目标是在任何情况下都会执行的目标，默认目标为初始状态下的默认目标，其他则是用来切换状态后执行的目标。

这里注册目标有两种方法

``` java
    private static void registerIdleGoals(Brain<MarisaEntity> brain) {
        Pair<Integer, BehaviorControl<? super MarisaEntity>> updateActivity = Pair.of(11, new MarisaUpdateActivity());
        Pair<Integer, BehaviorControl<? super MarisaEntity>> attack = Pair.of(12, StartAttacking.create((marisa) -> true, MarisaBrain::findNearestValidAttackTarget));

        brain.addActivity(Activity.IDLE, ImmutableList.of(updateActivity, attack));
    }
```

第一种是手动设置每一个目标的优先级，数值低的会优先执行，这里我们构建了一个装有多个`Behavior`的不可变列表，然后写入大脑的`Activity.IDLE`，这样在大脑处于IDEL状态时，会依次执行这里的行为。

``` java
    private static void registerAggressiveGoals(Brain<MarisaEntity> brain, MarisaEntity marisaEntity) {
        brain.addActivityAndRemoveMemoryWhenStopped(
                Activity.FIGHT,
                10,
                ImmutableList.<net.minecraft.world.entity.ai.behavior.BehaviorControl<? super MarisaEntity>>of(
                        StopAttackingIfTargetInvalid.create(living -> !isNearestValidAttackTarget(marisaEntity, living)),
                        SetWalkTargetFromAttackTargetIfTargetOutOfReach.create(1.0F),
                        MeleeAttack.create(20)
                ),
                MemoryModuleType.ATTACK_TARGET
        );
    }
```

第二种会自动按列表顺序给予优先级，这里我们设置了攻击行为。

在大脑中写入了`Activity`和对应的`ImmutableList<? extends BehaviorControl<? super E>>`后，我们设置了大脑的核心活动，设置默认活动并启用。这样，我们就完成了大脑的基础工作。

注意，如果你的`Behavior`中向大脑写入了没有在`getMemoryTypes`中的内容，会给出报错，需要加入对应的记忆种类。

当然，如果你对原版提供的Activity，SensorType不满足的话，可以自行注册对应的部分并使用，比如

``` java
public class ActivityRegistry {
    public static final DeferredRegister<Activity> ACTIVITY_TYPES = DeferredRegister.create(Registries.ACTIVITY, YOUR_MOD.MODID);
    public static Supplier<Activity> RELEASE_SKILL = ACTIVITY_TYPES.register("release_skill",() -> register("release_skill"));
    public static Supplier<Activity> RELEASE_MAGIC = ACTIVITY_TYPES.register("release_magic",() -> register("release_magic"));

    private static Activity register(String key) {
        return Registry.register(BuiltInRegistries.ACTIVITY, key, new Activity(key));
    }
}
```

而对于想自己设置执行的`Behavior`时，可以实现`BehaviorControl<E>`接口，或者继承自`Behavior<E extends LivingEntity>`。