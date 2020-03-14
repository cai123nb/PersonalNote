# Merge Sort: Counting Inversions

归并排序拓展, 统计倒置次数. 算法详情请查看`HackerRank`: [原文 link](https://www.hackerrank.com/challenges/ctci-merge-sort/problem)

## Description

In an array,`arr`, the elements at indices `i` and `j` (where `i < j`) form an inversion if `arr[i] > arr[j]`. In other words, inverted elements `arr[i]` and `arr[j]` are considered to be "out of order". To correct an inversion, we can swap adjacent elements.

For example, consider the dataset `arr = [2,4,1]`. It has two inversions: `(4,1)`and `(2,1)`. To sort the array, we must perform the following two swaps to correct the inversions:

`arr=[2,4,1] -> swap(arr[1], arr[2]) -> swap(arr[0], arr[1]) -> [1,2,4]`

Given `d` datasets, print the number of inversions that must be swapped to sort each dataset on a new line.

### Function Description

Complete the function countInversions in the editor below. It must return an integer representing the number of inversions required to sort the array.

countInversions has the following parameter(s):

- `arr`: an array of integers to sort .

### Input Format

The first line contains an integer, `d`, the number of datasets.

Each of the next `d` pairs of lines is as follows:

- The first line contains an integer, `n`, the number of elements in `arr`.

- The second line contains `n` space-separated integers, `arr[i]`.

### Constraints

- `1 <= d <= 15`

- `1 <= n <= 10^5`

- `1 <= arr[i] <= 10^7`

### Output Format

For each of the `d` datasets, return the number of inversions that must be swapped to sort the dataset

## Sample

### Sample Input

```java
2
5
1 1 1 2 2
5
2 1 3 1 2
```

### Sample Output

```java
0
4
```

### Explanation

We sort the following `d = 2` datasets:

- `arr = [1,1,1,2,2]` is already sorted, so there are no inversions for us to correct. Thus, we print `0` on a new line.

- `arr = [2,1,3,1,2] -(1swap)-> [1,2,3,1,2] -(2swap)-> [1,1,2,3,2] -(1swap)-> [1,1,2,2,3]` We performed a total of `1 + 2 + 1 = 4` swaps to correct inversions.

## Thinking space

归并排序拓展, 熟悉归并排序即可.

## Solution

```java
// Complete the countInversions function below.
static long countInversions(int[] arr) {
    long swaps = 0;
    for (int gap = 1; gap < arr.length; gap = 2 * gap) {
        //Merge two gag's length sorted array to a sorted array.
        swaps += mergePass(arr, gap, arr.length);
    }
    return swaps;
}

private static long mergePass(int[] array, int gap, int length) {
    long swaps = 0;
    int i = 0;
    // Merge two adjacent gap length's datas
    for (i = 0; i + 2 * gap - 1 < length; i = i + 2 * gap) {
        swaps += merge(array, i, i + gap - 1, i + 2 * gap - 1);
    }
    if (i + gap - 1 < length) {
        //If left datas' length is larger than a gap which means
        // the left datas need to sort. else the data is already sorted.
        swaps += merge(array, i, i + gap - 1, length - 1);
    }
    return swaps;
}

private static long merge(int[] array, int low, int mid, int high) {
    long swaps = 0;
    int i = low, j = mid + 1, k = 0;
    int[] array2 = new int[high - low + 1];
    while (i <= mid && j <= high) {  //Merge target number into array2
        if (array[i] <= array[j]) {
            array2[k] = array[i];
            i++;
            k++;
        } else {
            array2[k] = array[j];
            //because adjacent array is sorted, so the right
            //array swaps times max_value will be a gap (mid + 1)
            //If left array choose one to store, swaps times will
            //decrease one
            swaps += mid + 1 - i;
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
    return swaps;
}
```
