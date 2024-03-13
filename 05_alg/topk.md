## 排序求解
排序是最容易想到的方法，将n个数排序之后，取出最大的k个，即为所得。

## 局部排序

不再全局排序，只对最大的k个排序。

将全局排序优化为了局部排序，非TopK的元素是不需要排序的，节省了计算资源。不少朋友会想到，需求是TopK，是不是这最大的k个元素也不需要排序呢？这就引出了第三个优化方法。

## 堆
思路：只找到TopK，不排序TopK。

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/05_alg/topk-heap1.webp "图片title")

先用前k个元素生成一个小顶堆，这个小顶堆用于存储，当前最大的k个元素。

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/05_alg/topk-heap2.webp "图片title")

接着，从第k+1个元素开始扫描，和堆顶（堆中最小的元素）比较，如果被扫描的元素大于堆顶，则替换堆顶的元素，并调整堆，以保证堆内的k个元素，总是当前最大的k个元素。


![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/05_alg/topk-heap3.webp "图片title")

直到，扫描完所有n-k个元素，最终堆中的k个元素，就是猥琐求的TopK。