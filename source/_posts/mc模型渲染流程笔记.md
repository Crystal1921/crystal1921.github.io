---
title: mc模型渲染流程笔记
date: 2025-1-22 1:24:00
tags: 
  - mod
  - render
---

从调用链上看，mc在渲染物品的时候需要一个BakedModel，这个对象在`ItemRenderer`的`getModel`方法中返回

``` java
    public BakedModel getModel(ItemStack stack, @Nullable Level level, @Nullable LivingEntity entity, int seed) {
        BakedModel bakedmodel;
        if (stack.is(Items.TRIDENT)) {
            bakedmodel = this.itemModelShaper.getModelManager().getModel(TRIDENT_IN_HAND_MODEL);
        } else if (stack.is(Items.SPYGLASS)) {
            bakedmodel = this.itemModelShaper.getModelManager().getModel(SPYGLASS_IN_HAND_MODEL);
        } else {
            bakedmodel = this.itemModelShaper.getItemModel(stack);
        }

        ClientLevel clientlevel = level instanceof ClientLevel ? (ClientLevel)level : null;
        BakedModel bakedmodel1 = bakedmodel.getOverrides().resolve(bakedmodel, stack, clientlevel, entity, seed);
        return bakedmodel1 == null ? this.itemModelShaper.getModelManager().getMissingModel() : bakedmodel1;
    }
```

这里可以看到mc对三叉戟模型和望远镜模型做了特殊判定，下面会继续从`itemModelShaper`中获取模型，这里neoforge把它替换为了`RegistryAwareItemModelShaper`来进行处理。

``` java 
/**
 * Wrapper around {@link ItemModelShaper} that cleans up the internal maps to respect ID remapping.
 */
@ApiStatus.Internal
public class RegistryAwareItemModelShaper extends ItemModelShaper {
    private final Map<Item, ModelResourceLocation> locations = Maps.newIdentityHashMap();
    private final Map<Item, BakedModel> models = Maps.newIdentityHashMap();

    public RegistryAwareItemModelShaper(ModelManager manager) {
        super(manager);
    }

    @Override
    @Nullable
    public BakedModel getItemModel(Item item) {
        return models.get(item);
    }

    @Override
    public void register(Item item, ModelResourceLocation location) {
        locations.put(item, location);
        models.put(item, getModelManager().getModel(location));
    }
}
```

我们可以看到每个`Item`对应着一个`BakedModel`，而`ModelResourceLocation`与`BakedModel`的对应关系在`ModelManager`中声明。那么结论是，我们只需要把ModelManager中Map`<ModelResourceLocation, BakedModel> bakedRegistry`记录的对应数据覆写，即可实现替换目标模型。

这里，neoforge提供了一个事件ModelEvent.ModifyBakingResult，可以用来往models中写入新的或覆写键值对。

``` java 
@EventBusSubscriber(bus = EventBusSubscriber.Bus.MOD)
public class ModifyBakingEvent {
    @SubscribeEvent
    public static void onModelBakeEvent(ModelEvent.ModifyBakingResult event) {
        Map<ModelResourceLocation, BakedModel> models = event.getModels();
        models.put(ModelResourceLocation.inventory(getResourceLocation("block_bit")), new BitItemSmartModel());
    }
}
```

这样，我们就实现了偷天换日，直接在模型加载层面上替换为了实现了BakedModel接口的新类BitItemSmartModel。