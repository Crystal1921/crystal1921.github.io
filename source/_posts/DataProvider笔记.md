---
title: DataProvider笔记
date: 2025-5-25 23:00:00
tags: 
  - mod
  - java
  - mc
---

dataGen是一个很好用的功能，如果原版和neoforge提供的无法满足需求的时候，你可以自己实现DataProvider来创造自己的dataGen

```java
    CompletableFuture<?> run(CachedOutput var1);

    String getName();
```

DataProvider中有两个接口需要实现，其中run方法决定了生成的过程

```java
    public @NotNull CompletableFuture<?> run(@NotNull CachedOutput cache) {
        this.clearDialogs();
        this.registerDialogs();
        return this.generateAllDialogs(cache);
    }
```

在这个方法中，我们先清除缓存的部分，后通过抽象方法`registerDialogs`实现需要的数据的写入。然后开始生成

```java
    protected CompletableFuture<?> generateAllDialogs(CachedOutput cache) {
        CompletableFuture<?>[] futures = new CompletableFuture[dialogSequences.size()];
        int i = 0;

        DialogSequence dialog;
        Path target;
        for(Iterator<DialogSequence> iterator = dialogSequences.iterator();
            iterator.hasNext();
            futures[i++] = DataProvider.saveStable(cache, serializeDialog(dialog), target)) {

            dialog = iterator.next();
            target = this.getPath(dialog);
        }

        return CompletableFuture.allOf(futures);
    }

    protected Path getPath(DialogSequence dialogSequence) {
        return this.output
            .getOutputFolder(PackOutput.Target.DATA_PACK)
            .resolve(Dialog.MODID)
            .resolve("dialogs")
            .resolve(dialogSequence.getId() + ".json");
    }

    private JsonElement serializeDialog(DialogSequence dialog) {
        Gson gson = new GsonBuilder().setPrettyPrinting().create();
        return gson.toJsonTree(dialog);
    }
```

这里我们有两个工具方法，第一个getPath指定了我们生成的目标文件。在`data/modid/dialogs/`下面，生成一个json文件。

第二个工具方法负责把目标类的字段序列化为`JsonElement`。

在主方法`generateAllDialogs`中，我们把`dialogSequences`列表中的所有元素都通过这两个方法加入到`dataGen`的过程中。

这样，这个抽象的DataProvider就完成了，只需要继承这个类，并且在`registerDialogs()`方法中加入所要生成的DialogSequence对象，就完成了。