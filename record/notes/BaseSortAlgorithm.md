# 排序算法小结

排序一般分为两种: 内部排序和外部排序. 内部排序: 所有的数据记录在内存中进行排序, 而外部排序: 由于进行排序的数据过于庞大, 无法全部放入内存中, 需要借助外部存储, 多次进行数据交换, 以达到排序整个文件的目的.

## 内部排序算法

### 插入排序

#### 直接插入排序(Straight Insertion Sort)

**思路:** 选取一个元素作为排好序的组合, 每次将一个元素有顺序地插入到其中, 直到将所有的元素插入完毕.

![Show](https://image.cjyong.com/blog/r16.jpg)

代码实现:

```java
/**
     *  直接插入排序算法,
     *  时间复杂度:
     *  最好情况:O(n), 排好序的情况下, 每次都插入到最后一个.
     *  最差情况:O(n^2), 每次都插入到最前面, 1 + 2 + ... + n: O(n^2/2) = O(n^2)
     *  平均情况:O(n^2)
     * @param datas
     * @return
     */
    private static int[] straightInsertionSort(int[] datas) {
        int[] result = Arrays.copyOf(datas, datas.length);   //Store result
        for (int i = 1; i < result.length; i++) {
            if (result[i] < result[i - 1]) {                  //The Order Wrong
                int tmpValue = result[i];
                int position = i - 1;
                for(int j = position; j >= 0 && tmpValue < result[j]; j--)  //Find right position to insert
                {
                    result[j + 1] = result[j];
                    position--;
                }
                result[position + 1] = tmpValue;
            }
        }
        return  result;
    }
```

#### 希尔排序(Shell`s Sort)

**思路**: 希尔排序是把记录按下标的一定增量分组, 对每组使用直接插入排序算进行排序. 随着增量逐渐减少, 每组包含的关键词越来越多, 当增量减至 1 时, 整个文件恰被分成一组, 算法便终止.
提出的原因:

- 插入排序在对几乎已经排好序的数据操作时, 效率高, 即可以达到线性排序的效率.
- 插入排序在一般情况下是低效的, 因为插入排序每次只能将数据移动一位.

如, 对该数组进行排序: 49, 38, 65, 97, 76, 13, 27, 49, 55, 04.
增量的取值: 5 , 3, 1
过程为:

![Show](https://image.cjyong.com/blog/r17.jpg)

代码实现：

```java
/**
     * 希尔排序, 直接插入排序的一种改进, 不稳定的
     * 时间复杂度: 希尔排序时间复杂度依赖增量的取值
     * 最好情况:O(n^1.3)
     * 最坏情况: O(n^2)
     * 平均情况:O(nlogn) - O(n^2)
     *
     *
     * @param datas
     * @return
     */
    private static int[] shellSort(int[] datas) {
        int[] result = Arrays.copyOf(datas, datas.length);   //Store result
        int increment = datas.length;   //Increment define
        while (true) {
            increment = increment / 2;   //Default increment take half every time
            for (int x = 0; x < increment; x++) {
                for (int i = x + increment; i < datas.length; i = i + increment) {  //Implemented by straight insert sorting
                    int tmpValue = datas[i];
                    int position = 0;
                    for(position = i - increment; position >= 0 && result[position] > tmpValue; position = position - increment) {
                        result[position + increment] = result[position];
                    }
                    result[position + increment] = tmpValue;
                }
            }
            if (increment == 1) {
                break;
            }
        }
        return  result;
    }
```

## 交换排序

### 冒泡排序

**思路**：在要排序的一组数中, 对当前还未排好序的范围内的全部数, 自上而下对相邻的两个数依次进行比较和调整, 让较大的数往下沉, 较小的往上冒. 即: 每当两相邻的数比较后发现它们的排序与排序要求相反时, 就将它们互换.

![show](https://image.cjyong.com/blog/r18.jpg)

代码实现：

```java
/**
     * 冒泡排序， 稳定的
     * 时间复杂度:
     * 最好情况: O(n),
     * 最坏情况: O(n^2)
     * 平均情况: O(n^2)
     *
     * @param datas
     * @return
     */
    private static int[] bubbleSort(int[] datas) {
        int[] result = Arrays.copyOf(datas, datas.length);
        for (int i =0 ; i< datas.length; i++) {
            for(int j = 0; j < datas.length - i - 1; j++) {
                if(result[j] > result[j+1]) { //Swap
                    int tmp = result[j] ;
                    result[j] = result[j+1] ;
                    result[j+1] = tmp;
                }
            }
        }
        return result;
    }
    /**
     * 冒泡算法的改进，每次便利寻找最大值和最小值， 减少一半的遍历次数
     *
     * @param datas
     * @return
     */
    private static int[] bubbleSortImprove(int[] datas) {
        int[] result = Arrays.copyOf(datas, datas.length);
        int low = 0, high = datas.length - 1, j = 0, tmp = 0;
        while (low < high) {
            for (j = low; j < high; j++)  //Get Max value
                if (result[j] > result[j+1]) {
                    tmp = result[j] ;
                    result[j] = result[j+1] ;
                    result[j+1] = tmp;
                }
            --high;
            for ( j=high; j>low; j--)  //Get Mini Value
                if (result[j]<result[j-1]) {
                    tmp = result[j];
                    result[j]=result[j-1];
                    result[j-1]=tmp;
                }
            ++low;
        }
        return result;
    }
```

### 快速排序

**思路**： 假设对 a[p:r]进行排序

1. 分解：以 a[q]为基准元素将 a[p:r]分解成三段：a[p:q-1], a[q], a[q+1:r].其中 a[p:q-1]都是小于 a[q]的, a[q+1:r]都是大于 a[q]的.
2. 递归求解： 通过递归的调用快速排序算法分别对 a[p:q-1]和 a[q+1:r]进行排序.
3. 合并： 由于对 a[p:q-1],a[q]和 a[q+1:r]本身就是有序的, 所以 a[p:r]就是已经拍好序的.

![show](https://image.cjyong.com/blog/r20.jpg)

代码实现：

```java
/**
     * 快速排序算法， 不稳定的
     * 时间复杂性:
     * 最好情况: O(nlogn)
     * 最差情况: O(n^2)
     * 平均情况: O(nlogn)
     *
     *
     * @param datas
     * @param start
     * @param end
     */
    private static void quickSort(int[] datas, int start, int end) {
        if (start < end) {
            int splitPoint = partition(datas, start, end);    //Swap number between splitPoint
            quickSort(datas, start, splitPoint - 1);    //Sort left side
            quickSort(datas, splitPoint + 1, end);      //Sort right side
        }
    }

    /**
     * Swap the data between splitPoint, make sure the numbers
     * between start : splitPoint is less than splitPoint's number
     * the numbers between splitPoint : end is larger than splitPoint's number.
     *
     * @param datas
     * @param start
     * @param end
     * @return
     */
    private static int partition(int[] datas, int start, int end) {
        int i = start, j = end + 1, x = datas[start];   //Here use datas[start] as split Point
        while (true) {
            while (datas[++i] < x && i < end);
            while (datas[--j] > x);
            if(i >= j) break;
            int tmp = datas[i]; //Swap two values.
            datas[i] = datas[j];
            datas[j] = tmp;
        }
        datas[start] = datas[j];
        datas[j] = x;
        return j;
    }
```

## 选择排序

### 简单选择排序(Simple Selection Sort)

**思路**： 在要排序的一组数中, 选出最小(或者最大)的一个数与第 1 个位置的数交换; 然后在剩下的数当中再找最小(或者最大)的与第 2 个位置的数交换, 依次类推, 直到第 n-1 个元素(倒数第二个数)和第 n 个元素(最后一个数)比较为止.

![show](https://image.cjyong.com/blog/r19.jpg)

代码实现：

```java
/**
 * 简单选择排序， 稳定的
 * 时间复杂度:
 * 最好情况: O(n^2)
 * 最差情况: O(n^2)
 * 平均情况: O(n^2)
 *
 * @param datas
 * @return
 */
private static int[] simpleSelectSort(int[] datas) {
    int[] result = Arrays.copyOf(datas, datas.length);
    for (int i = 0; i < result.length; i++) {
        for (int j = i + 1; j < result.length; j++){
            if(result[i] > result[j]) {   //Swap
                int tmp = result[i];
                result[i] = result[j];
                result[j] = tmp;
            }
        }
    }
    return result;
}
```

### 堆排序(Heap Sort)

**思路**：堆排序是利用堆的性质进行的一种选择排序.

- n 个元素的序列{k1, k2,…,kn}当且仅当满足下列关系之一时, 称之为堆.
  - 情形 1：ki <= k2i 且 ki <= k2i+1 （最小化堆或小顶堆）
  - 情形 2：ki >= k2i 且 ki >= k2i+1 （最大化堆或大顶堆）
    基本方法： 我们可以确定堆顶的元素一定是最值, 初次使用时, 我们先初始化堆, 然后, 我们取出第一个元素, 从末尾选取一个元素放到堆顶, 初始化堆, 在取出一个元素, ... ,以此类推, 直到所有的元素取完. 是对直接选择排序的优化, 通过利用堆(类似二叉树)的特性将每次选择所产生的排序特征存储下来, 用于后面进行比较. 所以时间复杂度总是为: O(nlogn), 这是比快速选择法好的一个方面.

![show](https://image.cjyong.com/blog/r22.jpg)

代码实现:

```java
/**
     * 堆排序, 利用堆的特性进行选择排序 不稳定
     * 时间复杂度：
     * 最好,最坏,平均: O(nlogn)
     *
     * @param datas
     */
    private static void heapSort(int[] datas) {
        buildHeap(datas);   //Init array
        for ( int i = datas.length - 1; i >= 0; i--) {
            int tmp = datas[0];     //Swap root with last one
            datas[0] = datas[i];
            datas[i] = tmp;
            heapAdjust(datas, 0, i - 1);
        }
    }

    private static void buildHeap(int[] datas) {
        int size = datas.length - 1;
        for (int i = size / 2; i >= 0; i--) {
            heapAdjust(datas, i, size);
        }
    }

    private static void heapAdjust(int[] data, int i, int size) {
        int lChild = 2 * i + 1;
        int rChild = 2 * i + 2;
        int max = i;
        if( i <= size / 2) {
            if (lChild <= size && data[lChild] > data[max]) {
                max = lChild;
            }
            if (rChild <= size && data[rChild] > data[max]) {
                max = rChild;
            }
            if (max != i) {  //Current max value is not at root
                int tmp = data[i];  //Swap
                data[i] = data[max];
                data[max] = tmp;
                heapAdjust(data, max, size);    //Make sure the struct is right
            }
        }
    }
```

## 归并排序(Merge Sort)

**思路**: 利用分治策略对数组进行排序, 先对两个大小相同的数组进行排序, 合并成一个数组. 将已有序的子序列合并, 得到完全有序的序列; 即先使每个子序列有序, 再使子序列段间有序. 若将两个有序表合并成一个有序表, 称为二路归并.

![show](https://image.cjyong.com/blog/r23.jpg)

代码实现:

```java
/**
     * 归并排序算法, 稳定的
     *
     * 时间复杂度:
     * 最好,最坏,平均: O(nlogn)
     * @param datas
     */
    private static void mergeSort(int[] datas) {
        for (int gap = 1; gap < datas.length; gap = 2 * gap) {
            mergePass(datas, gap, datas.length);     //Merge two gag's length sorted array to a sorted array.
        }
    }

    private static void mergePass(int[] array, int gap, int length) {
        int i = 0;
        for (i = 0; i + 2 * gap - 1 < length; i = i + 2 * gap) {    //Merge two adjacent gap length's datas
            merge(array, i, i + gap - 1, i + 2 * gap - 1);
        }
        if (i + gap - 1 < length) {
            //If left datas' length is larger than a gap which means
            // the left datas need to sort. else, the data is already sorted.
            merge(array, i, i + gap - 1, length - 1);
        }
    }

    private static void merge(int[] array, int low, int mid, int high) {
        int i = low, j = mid + 1, k = 0;
        int[] array2 = new int[high - low + 1];
        while (i <= mid && j <= high) {  //Merge target number into array2
            if (array[i] <= array[j]) {
                array2[k] = array[i];
                i++;
                k++;
            } else {
                array2[k] = array[j];
                j++;
                k++;
            }
        }
        while (i <= mid) { //Left array still have some datas.
            array2[k] = array[i];
            i++;
            k++;
        }
        while (j <= high) { //Right array still have some datas.
            array2[k] = array[j];
            j++;
            k++;
        }
        for (k = 0, i = low; i <= high; i++, k++) { //Store result
            array[i] = array2[k];
        }
    }
```

## 桶排序

**思路**: 先对数据进行分组(大小或者关键码), 分组之后在对组内数据进行排序(快速, 归并等), 然后按照桶的顺序依次输出. 先分组, 在排序. 避免了受到比较排序极限(nlogn)影响. 最优化情况达到 O(N). 对空间的要求非常的大, 牺牲空间换取时间.

![show](https://image.cjyong.com/blog/r24.jpg)

简单代码实现:

```java
/**
     * 简单桶排序算法, 对空间的要求极大, 牺牲空间换时间
     * 时间复杂度: 对于n个数据, m个桶
     * 最好情况: O(n)
     * 平均情况: O(n+c) C=n*(logn-logm)
     *
     *
     * @param datas
     * @return
     */
    private static int[] bucketSort(int[] datas) {
        //Get max value
        int max = Integer.MIN_VALUE;
        for (int i : datas) {
            if (i > max) {
                max = i;
            }
        }
        int[] buckets = new int[max + 1];
        for (int data : datas) {
            buckets[data]++;
        }
        //Get Result
        int[] result = new int[datas.length];
        int i = 0;
        for (int j = 0; j < buckets.length; j++) {
            int tmp = buckets[j];
            if (tmp != 0) {
                while(tmp > 0) {
                    result[i++] = j;
                    tmp--;
                }
            }
        }
        return result;
    }
```

## 拓展

### 随机选择的问题

在现实生活中我们, 有时候并不需要对所有的数据进行排序, 我们只是简单的想获取其中第 N 大的一个值, 比如我想获取这个班上数学考试第三名是谁, 我并不需要所有人的排序. 当 N = 1 时, 就是我要求的最大值, N = n 时就是最小值, N = (1 + n) / 2 时, 就是中位数. 求最大值和最小值, 我们很容易得到时间复杂度为 O(n), 当我们求第 N 大是, 感觉就会难了很多. 如我们要获取中位数明显比最大值最小值难很多. 但是如果我们从渐进阶的角度来考虑问题, 它们都是一样的, 一般选择的问题都能在 O(n)的复杂度解决.

```java
  /**
     * 从data中选择第k大的数据
     *
     * @param data 数据
     * @param p 开始点, 初始情况请赋值0
     * @param r 结束点, 初始情况请赋值最大值
     * @param k 第k大
     * @return
     */
    private static int randomizedSelect(int[] data, int p, int r, int k) {
        if (p == r) return data[p];
        int i = randomizedPartition(data, p, r), j = i - p + 1;
        if (k <= j)
            return randomizedSelect(data, p ,i , k);
        else
            return randomizedSelect(data, i + 1, r, k - j);
    }

    /**
     * 改方法可用于快速选择的优化
     *
     * @param data
     * @param p
     * @param r
     * @return
     */
    private static int randomizedPartition(int[] data, int p, int r) {
        Random random = new Random(System.currentTimeMillis());
        int i = random.nextInt(r -  p) + r;
        return partition(data, p, r);
    }
```

## 总结

![show](https://image.cjyong.com/blog/r25.jpg)

时间复杂度排序:
O(n^2): 直接插入,直接选择和冒泡排序
O(nlogn): 快速,堆,归并
O(n^1.3):希尔排序(取决于增量的取值)
O(n): 桶排序,基数排序...
稳定性:

> 排序算法的稳定性: 若待排序的序列中, 存在多个具有相同关键字的记录, 经过排序, 这些记录的相对次序保持不变, 则称该算法是稳定的; 若经排序后, 记录的相对次序发生了改变.则称该算法是不稳定的.

稳定的排序算法： 冒泡排序,插入排序,归并排序和基数排序

不是稳定的排序算法：选择排序,快速排序,希尔排序,堆排序

## 相关阅读

[真实归属的博客](http://blog.csdn.net/hguisu/article/details/7776068/)
