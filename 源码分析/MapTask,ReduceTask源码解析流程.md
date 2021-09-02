## MapTask
```
context.write(k, NullWritable.get()); //自定义的 map 方法的写出，进入
output.write(key, value);
//MapTask727 行，收集方法，进入两次
collector.collect(key, value,partitioner.getPartition(key, value, partitions));
HashPartitioner(); //默认分区器
collect() //MapTask1082 行 map 端所有的 kv 全部写出后会走下面的 close 方法
close() //MapTask732 行
collector.flush() // 溢出刷写方法，MapTask735 行，提前打个断点，进入
sortAndSpill() //溢写排序，MapTask1505 行，进入
sorter.sort() QuickSort //溢写排序方法，MapTask1625 行，进入
mergeParts(); //合并文件，MapTask1527 行，进入
collector.close(); //MapTask739 行,收集器关闭,即将进入 ReduceTask
```

## ReduceTask
```
if (isMapOrReduce()) //reduceTask324 行，提前打断点
initialize() // reduceTask333 行,进入
init(shuffleContext); // reduceTask375 行,走到这需要先给下面的打断点
 totalMaps = job.getNumMapTasks(); // ShuffleSchedulerImpl 第 120 行，提前打断点
 merger = createMergeManager(context); //合并方法，Shuffle 第 80 行
// MergeManagerImpl 第 232 235 行，提前打断点
this.inMemoryMerger = createInMemoryMerger(); //内存合并
this.onDiskMerger = new OnDiskMerger(this); //磁盘合并
rIter = shuffleConsumerPlugin.run();
eventFetcher.start(); //开始抓取数据，Shuffle 第 107 行，提前打断点
eventFetcher.shutDown(); //抓取结束，Shuffle 第 141 行，提前打断点
copyPhase.complete(); //copy 阶段完成，Shuffle 第 151 行
taskStatus.setPhase(TaskStatus.Phase.SORT); //开始排序阶段，Shuffle 第 152 行
sortPhase.complete(); //排序阶段完成，即将进入 reduce 阶段 reduceTask382 行
reduce(); //reduce 阶段调用的就是我们自定义的 reduce 方法，会被调用多次
cleanup(context); //reduce 完成之前，会最后调用一次 Reducer 里面的 cleanup 方法
```