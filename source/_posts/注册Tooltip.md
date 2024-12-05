---
title: 注册Tooltip
date: 2024-12-5 23:00:00
tags: 
  - mod
  - java
  - mc
---

首先我们需要一个类实现TooltipComponent接口，这个类的初始化接受我们需要渲染传递的参数，简单的，这里放上一个record类就好了

```java
public record RecordDanmakuTooltip(ArrayList<Point> pointArrayList) implements TooltipComponent {

}
```

然后就是客户端上的渲染类，实现ClientTooltipComponent接口

```java
public class ClientDanmakuTooltip implements ClientTooltipComponent {

    private final ArrayList<Point> danmakuList;

    public ClientDanmakuTooltip(RecordDanmakuTooltip recordDanmakuTooltip) {
        this.danmakuList = recordDanmakuTooltip.pointArrayList();
    }

    @Override
    public int getHeight() {
        return 120;
    }

    @Override
    public int getWidth(@NotNull Font font) {
        return 120;
    }

    @Override
    public void renderImage(@NotNull Font font, int pX, int pY, @NotNull GuiGraphics pGuiGraphics) {

    }
}

```

这里前两个方法写明了该Tooltip所占的面积，在renderImage中就可以按自己的要求绘制图案了

这里Forge提供了事件，让我们注册自己的Tooltip

```java
@Mod.EventBusSubscriber(value = Dist.CLIENT, bus = Mod.EventBusSubscriber.Bus.MOD)
public class TooltipRegistry {
    @SubscribeEvent
    public static void onRegisterClientTooltip(RegisterClientTooltipComponentFactoriesEvent event) {
        event.register(RecordDanmakuTooltip.class,ClientDanmakuTooltip::new);
    }
}

```

最后，就可以在物品中覆写getTooltipImage方法来放上自己的Tooltip

```java
    @Override
    public @NotNull Optional<TooltipComponent> getTooltipImage(@NotNull ItemStack itemstack) {
        if (itemstack.getOrCreateTag().get("crystal_point") != null){
            return Optional.of(new RecordDanmakuTooltip(pointArrayList));
        }
        return Optional.empty();
    }

```