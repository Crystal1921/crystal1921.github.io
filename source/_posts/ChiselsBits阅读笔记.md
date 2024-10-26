---
title: ChiselsBits阅读笔记
tags: 
  - java
  - mod
  - mc
---

梳理一下最近阅读雕凿工艺Chisels & Bits的工作方法

对于任意雕刻方块来说，渲染线程会尝试拿到他的模型。
方块实体是DataAwareChiseledBlockBakedModel类的getModelData方法。
物品是DataAwareChiseledBlockBakedModel类的匿名内部类，覆写ItemOverrides的resolve方法

这两个最后都指向ChiseledBlockSmartModel类的getOrCreateBakedModel方法，这个方法的作用就是拿到缓存中已有的模型，如果没有就生成，烘焙一个模型。

getOrCreateBakedModel有很多重载方法，目的是从物品NBT或者方块实体的数据中拿到用于储存雕刻数据的VoxelBlob对象，把这个对象与缓存中的进行比对，如果有就直接获取，如果没有就构建一个新的ChiseledBlockBakedModel对象，并写入缓存。

在初始化ChiseledBlockBakedModel对象中，根据传入的VoxelBlob体素数据，在generateFaces中构建模型并烘焙。
首先分别按照体素数据在三个方向上构建面，然后合并可以合并的面，再写入顶点数据，烘焙成BakedQuad
最终建立一个体素数据和对应的烘焙模型的，写进缓存，避免多次烘焙同一个模型

VoxelBlob本质上是一个BitSet(4096)，也就是16^3，用来储存一个方块里所有小方块的数据。通过如下方法固定hash值，并覆写判断方法，来实现判断两个VoxelBlob对象中的数据是否相同，保证缓存中不会出现相同的VoxelBlob对象

``` java
    @Override
    public boolean equals(final Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof VoxelBlob)) {
            return false;
        }
        final VoxelBlob voxelBlob = (VoxelBlob) o;
        return detail == voxelBlob.detail && Arrays.equals(values, voxelBlob.values);
    }

    @Override
    public int hashCode() {
        int result = Objects.hash(detail);
        result = 31 * result + Arrays.hashCode(values);
        return result;
    }
```

之前碰到的很多bug是由ClientSide的showGhost方法引起的，这个方法的作用是渲染一个预览的方块，但是这个方法中只往缓存中写了VoxelBlobStateReference，但是没有调用getOrCreateBakedModel生成对应的烘焙模型，而且其他体素数据相同的模型会在缓存中拿到一个空面的模型，并不会尝试生成模型，导致没有正确渲染。
后来尝试了很多办法，比如在中键获取方块模型的时候提起烘焙四个旋转方向的模型，但是由于在渲染时序中，renderLevel要比renderItemInHand先执行，还是会有相同的问题，治标不治本，最后只能修复showGhost方法。