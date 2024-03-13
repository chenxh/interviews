## 排序求解
排序是最容易想到的方法，将n个数排序之后，取出最大的k个，即为所得。

## 局部排序

不再全局排序，只对最大的k个排序。

将全局排序优化为了局部排序，非TopK的元素是不需要排序的，节省了计算资源。不少朋友会想到，需求是TopK，是不是这最大的k个元素也不需要排序呢？这就引出了第三个优化方法。

## 堆
思路：只找到TopK，不排序TopK。

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/05_alg/imgs/topk-heap1.webp "图片title")

先用前k个元素生成一个小顶堆，这个小顶堆用于存储，当前最大的k个元素。

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/05_alg/imgs/topk-heap2.webp "图片title")

接着，从第k+1个元素开始扫描，和堆顶（堆中最小的元素）比较，如果被扫描的元素大于堆顶，则替换堆顶的元素，并调整堆，以保证堆内的k个元素，总是当前最大的k个元素。


![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/05_alg/imgs/topk-heap3.webp "图片title")

直到，扫描完所有n-k个元素，最终堆中的k个元素，就是猥琐求的TopK。
时间复杂度：O(n*lg(k))

大数据量求 top k 时适用。


## 随机选择

分治法（Divide&Conquer），把一个大的问题，转化为若干个子问题（Divide），每个子问题“都”解决，大的问题便随之解决（Conquer）。这里的关键词是“都”。从伪代码里可以看到，快速排序递归时，先通过partition把数组分隔为两个部分，两个部分“都”要再次递归。例如：快速排序

分治法有一个特例，叫减治法。

减治法（Reduce&Conquer），把一个大的问题，转化为若干个子问题（Reduce），这些子问题中“只”解决一个，大的问题便随之解决（Conquer）。这里的关键词是“只”。例如：二分查找， 随机选择

TopK是希望求出arr[1,n]中最大的k个数，那如果找到了第k大的数，做一次partition，不就一次性找到最大的k个数了么？

partition会把整体分为两个部分。

更具体的，会用数组arr中的一个元素（默认是第一个元素t=arr[low]）为划分依据，将数据arr[low, high]划分成左右两个子数组：

左半部分，都比t大

右半部分，都比t小

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/05_alg/imgs/topk-select.webp.webp "图片title")


***通过随机选择（randomized_select），找到arr[1, n]中第k大的数，再进行一次partition，就能得到TopK的结果。***

## 总结
全局排序，O(n*lg(n))

局部排序，只排序TopK个数，O(n*k)

堆，TopK个数也不排序了，O(n*lg(k))

分治法，每个分支“都要”递归，例如：快速排序，O(n*lg(n))

减治法，“只要”递归一个分支，例如：二分查找O(lg(n))，随机选择O(n)

TopK的另一个解法：随机选择+partition


