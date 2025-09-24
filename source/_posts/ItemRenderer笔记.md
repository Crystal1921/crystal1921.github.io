---
title: ItemRenderer笔记
date: 2025-9-23 23:00:00
tags: 
  - mod
  - render
---

## 渲染流程

``` java
    public void renderStatic(ItemStack arg, ItemDisplayContext arg2, int i, int j, PoseStack arg3, MultiBufferSource arg4, @Nullable Level arg5, int k) {
        this.renderStatic((LivingEntity)null, arg, arg2, false, arg3, arg4, arg5, i, j, k);
    }

    public void renderStatic(@Nullable LivingEntity arg, ItemStack arg2, ItemDisplayContext arg3, boolean bl, PoseStack arg4, MultiBufferSource arg5, @Nullable Level arg6, int i, int j, int k) {
        if (!arg2.isEmpty()) {
            BakedModel bakedmodel = this.getModel(arg2, arg6, arg, k);
            this.render(arg2, arg3, bl, arg4, arg5, i, j, bakedmodel);
        }

    }

    public BakedModel getModel(ItemStack arg, @Nullable Level arg2, @Nullable LivingEntity arg3, int i) {
        BakedModel bakedmodel;
        if (arg.is(Items.TRIDENT)) {
            bakedmodel = this.itemModelShaper.getModelManager().getModel(TRIDENT_IN_HAND_MODEL);
        } else if (arg.is(Items.SPYGLASS)) {
            bakedmodel = this.itemModelShaper.getModelManager().getModel(SPYGLASS_IN_HAND_MODEL);
        } else {
            bakedmodel = this.itemModelShaper.getItemModel(arg);
        }

        ClientLevel clientlevel = arg2 instanceof ClientLevel ? (ClientLevel)arg2 : null;
        BakedModel bakedmodel1 = bakedmodel.getOverrides().resolve(bakedmodel, arg, clientlevel, arg3, i);
        return bakedmodel1 == null ? this.itemModelShaper.getModelManager().getMissingModel() : bakedmodel1;
    }
```

我们从渲染调用链一步步向下看，会在`renderStatic`中调用`getModel`方法以获取模型，这里mc对三叉戟和望远镜进行了特判。让我们看看mc中三叉戟的json文件

``` json
{
    "parent": "builtin/entity",
    "gui_light": "front",
    "textures": {
        "particle": "item/trident"
    },
    "overrides": [
        {
            "predicate": {
                "throwing": 1
            },
            "model": "item/trident_throwing"
        }
    ]
}
```
这里发现他的父类型是`builtin/entity`，意味着它`getModel`返回的将是一个`BuiltInModel`对象，这是一个占位符空模型，意味着在以下的渲染流程中会特殊判别，其中`isCustomRenderer`默认返回`true`

接下来对普通的物品，调用`getItemModel`获取一个一般的物品模型，然后看看模型中有没有`overrides`字段，对当前模型进行替换，比如如上的三叉戟中，当符合`throwing`条件时，替换为`item/trident_throwing`模型，最后，如果前面都获取失败了，返回默认的紫黑块模型。

当然如果想要替换原版的紫黑块，可以mixin这个`getMissingModel()`方法

看起来mc只对三叉戟和望远镜进行了特判，但是在`net.minecraft.client.renderer.entity.ItemRenderer#renderStatic`中的下一步，在`render`方法中，麻将再次展现了他无与伦比的硬编码技术

在neoforge对`render`方法进行了patch，加入了`net.neoforged.neoforge.client.extensions.IBakedModelExtension#getRenderPasses`以便开发者可以自定义多层级模型。

``` java

    boolean flag = displayContext == ItemDisplayContext.GUI || displayContext == ItemDisplayContext.GROUND || displayContext == ItemDisplayContext.FIXED;

    if (!p_model.isCustomRenderer() && (!itemStack.is(Items.TRIDENT) || flag)) {
        //常规渲染
    }

    IClientItemExtensions.of(itemStack).getCustomRenderer().renderByItem(itemStack, displayContext, poseStack, bufferSource, combinedLight, combinedOverlay);
```

这里我们发现麻将再次对三叉戟进行了特判，加入到了`CustomRender`的行列中，在特殊的`CustomRender`中，将不返回模型回正常渲染流程，而是直接将模型烘焙完成后上传。

在`net.minecraft.client.renderer.BlockEntityWithoutLevelRenderer#renderByItem`在我们可以看到一大堆各种各样的硬编码特判，麻将你赢了。