---
title: HBase 1.1.2 split 策略
date: 2015-11-01 16:48:05
tags: [HBase]
categories: [大数据]
---
### 背景：

今天同事用ycsb做HBase的性能测试，反馈说reigon总是在配置的大小前split(配置的是10G)，于是我就给他说起了hbase的spilt策略：从0.94增加了新的策略，还是在会每次flush的时候会去判断需不需要split，但是判断的策略有了改变，会比较现有文件的大小与改表region个数的平方*memstore大小的关系，如果前者较大也会去做split......他打断说：region个数的平方*memstore这个不对吧，日志里是memstore二倍的大小，还给我截图为证，无奈，就翻看了1.1.2的代码，发下算法真的变了，对应变成了region个数的三次方*2*memstore。

日志如下：
````
regionserver.IncreasingToUpperBoundRegionSplitPolicy: ShouldSplit because 0 size=296442912, sizeToCheck=268435456, regionsWithCommonTable=1
````
从日志中可以看到，每次flush时候都会调用IncreasingToUpperBoundRegionSplitPolicy类中的shouldSplit方法，方法内容如下：

````java
 @Override
  protected boolean shouldSplit() {
    if (region.shouldForceSplit()) return true;
    boolean foundABigStore = false;
    // Get count of regions that have the same common table as this.region
    int tableRegionsCount = getCountOfCommonTableRegions();
    // Get size to check
    long sizeToCheck = getSizeToCheck(tableRegionsCount);

    for (Store store : region.getStores()) {
      // If any of the stores is unable to split (eg they contain reference files)
      // then don't split
      if ((!store.canSplit())) {
        return false;
      }

      // Mark if any store is big enough
      long size = store.getSize();
      if (size > sizeToCheck) {
        LOG.debug("ShouldSplit because " + store.getColumnFamilyName() +
          " size=" + size + ", sizeToCheck=" + sizeToCheck +
          ", regionsWithCommonTable=" + tableRegionsCount);
        foundABigStore = true;
      }
    }

    return foundABigStore;
  }
````

代码说明：

首先，通过getCountOfCommonTableRegions()方法获取目前的region个数tableRegionCount，然后通过getSizeTOCheck(tableRegionCount)方法运算得出一个阈值sizeToCheck,接着在for循环中遍历该reigon下所有的sotre，如果有store不能做split(调用HStore类的canSplit方法，该方法判断store下的hfile是否有被reference的，即region刚拆分，但hfile还处于reference状态，未完成拆分)，直接返回false。如果该store可以做split，则比较store下hfile的大小与sizeToCheck的值，如果大于则标识foundABigStore置为true。

接着看下getSizeTOCheck(tableRegionCount)方法：
````
  protected long getSizeToCheck(final int tableRegionsCount) {
    // safety check for 100 to avoid numerical overflow in extreme cases
    return tableRegionsCount == 0 || tableRegionsCount > 100 ? getDesiredMaxFileSize():
      Math.min(getDesiredMaxFileSize(),
        this.initialSize * tableRegionsCount * tableRegionsCount * tableRegionsCount);
  }
````

代码说明：

如果tableRegionCount的值是0或者大于100，则通过getDesiredMaxFileSize()方法读取配置文件中的hbase.hregion.max.filesize值(即前文说的10G)，否则进行Math.min判断，后面tableRegionCount三次方很容易理解，看看initialSize怎么来的，相关方法内容如下：

````

@Override
  protected void configureForRegion(HRegion region) {
    super.configureForRegion(region);
    Configuration conf = getConf();
    this.initialSize = conf.getLong("hbase.increasing.policy.initial.size", -1);
    if (this.initialSize > 0) {
      return;
    }
    HTableDescriptor desc = region.getTableDesc();
    if (desc != null) {
      this.initialSize = 2*desc.getMemStoreFlushSize();
    }
    if (this.initialSize <= 0) {
      this.initialSize = 2*conf.getLong(HConstants.HREGION_MEMSTORE_FLUSH_SIZE,
        HTableDescriptor.DEFAULT_MEMSTORE_FLUSH_SIZE);
    }
  }

````
代码说明：

代码逻辑也是非常简单，这里不再赘述。

补充，昨天以为getCountOfCommonTableRegions()的逻辑是获取这张表所有的region，今天测试发现不是这样，回头再看代码，代码内容如下：

````
  private int getCountOfCommonTableRegions() {
    RegionServerServices rss = this.region.getRegionServerServices();
    // Can be null in tests
    if (rss == null) return 0;
    TableName tablename = this.region.getTableDesc().getTableName();
    int tableRegionsCount = 0;
    try {
      List<Region> hri = rss.getOnlineRegions(tablename);
      tableRegionsCount = hri == null || hri.isEmpty()? 0: hri.size();
    } catch (IOException e) {
      LOG.debug("Failed getOnlineRegions " + tablename, e);
    }
    return tableRegionsCount;
  }
}

````
代码说明:

首先获取该region所在的regionserver，然后获取该regionserver上的所有region，而不是该表在整个集群中的region数量。
