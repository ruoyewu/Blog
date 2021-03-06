---
title: 算法-排序
date: 2018-04-02 15:00
tags:
	- algorithm
---

排序是最最基本的算法之一，一般的排序算法有以下几种：

1. 插入排序
2. 选择排序
3. 冒泡排序
4. 快速排序
5. 堆排序
6. 希尔排序
7. 归并排序
8. 计数排序
9. 桶排序
10. 基数排序

## 插入排序

从数组第一个开始，根据当前需要排序的位置的值，找到在它之前的一个有序数组里面，它应该在的位置，然后将这个位置之后的值一次后移一位，再将这个值放入这个位置。

时间复杂度$O(n^2)$

空间复杂度$O(1)$

```java
public class InsertSort {
    public static void sort(int[] data) {
        for (int i = 0; i < data.length; i++) {
            int j = i;
            int target = data[j];
            
            // 顺位后移
            while (j > 0 && data[j - 1] > target) {
                data[j] = data[j - 1];
                j --;
            }
            
            // 插入
            data[j] = target;
        }
    }
}
```

## 选择排序

从数组第一个开始，从要排序的位置之后找到一个最小的值，将那个值与当前位置的值互换位置，然后开始找下一位。

时间复杂度$O(n^2)$

空间复杂度$O(1)$

```java
public class SelectSort {
    public static void sort(int[] data) {
        for (int i = 0; i < data.length; i++) {
            int min = i;
            
            // 找到最小的值的位置
            for (int j = i + 1; j < data.length; j++) {
                if (data[min] > data[j]) {
                    min = j;
                }
            }
            
            // 与当前位置互换
            if (min != i) {
                int temp = data[i];
                data[i] = data[min];
                data[min] = temp;
            }
        }
    }
}
```

## 冒泡排序

从数组的第一个开始，每次从数组的最后一位往前数，数到这一位，每次比较相邻的两位的大小，并将小的放到前面。

时间复杂度$O(n^2)$

空间复杂度$O(1)$

```java
public class BubbleSort {
    public static void sort(int[] data) {
        for (int i = 0; i < data.length; i++) {
            for (int j = data.length - 1; j > i ; j--) {
                
                // 如果靠后的值小于靠前的值，互换
                if (data[j] < data[j - 1]) {
                    int temp = data[j];
                    data[j] = data[j - 1];
                    data[j - 1] = temp;
                }
            }
        }
    }
}
```

## 快速排序

二分法原理，通过取一个值作为基准值，将所有大于这个值和小于这个值的数分别放在右边和左边，完成一半的排序，即每次找到一个中间数，然后将这两半各自再完成其一半的排序，直到排序完成。

时间复杂度$O(nlgn)$

空间复杂度$O(1)$

```java
public class QuickSort {
    public static void sort(int[] data) {
        int left = 0, right = data.length - 1;

        quickSort(data, left, right);
    }

    private static void quickSort(int[] data, int left, int right) {
        if (left >= right) {
            return;
        }
        int pivotPos = partition(data, left, right);
        quickSort(data, left, pivotPos - 1);
        quickSort(data, pivotPos + 1, right);
    }

    // 常规版
    private static int partition(int[] data, int left, int right) {
        int pivotKey = data[left];
        int pivotPointer = left;
        while (left < right) {
            while (left < right && data[right] >= pivotKey) {
                right --;
            }
            while (left < right && data[left] <= pivotKey) {
                left ++;
            }
            swap(data, left, right);
        }
        swap(data, pivotPointer, left);
        return left;
    }

    // 改进版
    private static int partition2(int[] data, int left, int right) {
        int pivotKey = data[left];

        while (left < right) {
            while (left < right && data[right] >= pivotKey) {
                right --;
            }
            data[left] = data[right];
            while (left < right && data[left] <= pivotKey) {
                left ++;
            }
            data[right] = data[left];
        }
        data[left] = pivotKey;
        return left;
    }

    private static void swap(int[] data, int l, int r) {
        int temp = data[l];
        data[l] = data[r];
        data[r] = temp;
    }
}
```

## 堆排序

借助堆实现的选择排序，利用堆的特性，每次将当前组中最大的一个值放到堆顶然后取走，接着将剩余的组中最大的一个值放到堆顶，直到只剩下一个数。

时间复杂度$O(nlgn)$

空间复杂度$O(1)$

```java
public class HeapSort {
    public static void sort(int[] data) {
        for (int i = data.length / 2; i >= 0; i --) {
            heapAdjust(data, i, data.length - 1);
        }

        for (int i = data.length - 1; i >= 0; i--) {
            swap(data, 0, i);
            heapAdjust(data, 0, i - 1);
        }
    }

    private static void heapAdjust(int[] data, int start, int end) {
        int temp = data[start];

        for (int i = 2 * start + 1; i <= end; i = 2 * i + 1) {
            if (i < end && data[i] < data[i + 1]) {
                i ++;
            }
            if (temp >= data[i]) {
                break;
            }
            data[start] = data[i];
            start = i;
        }

        data[start] = temp;
    }

    private static void swap(int[] data, int f, int t) {
        int temp = data[f];
        data[f] = data[t];
        data[t] = temp;
    }
}
```

## 希尔排序

插入排序的一种高效实现，也叫**缩小增量排序**，即先将数组变成大致有序，然后进行一遍插入排序，将数组变得大致有序的方法，就是「缩小增量排序」，即每次只排序数组的一部分，然后逐渐增加这个「一部分」，直到这个「一部分」等于整个数组，就变成了插入排序。

时间复杂度$O(n^{1.3})$

空间复杂度$O(1)$

```java
public class ShellSort {
    public static void sort(int[] data) {
        int d = data.length / 2;
        while (d >= 1) {
            shellInsert(data, d);
            d /= 2;
        }
    }

    private static void shellInsert(int[] data, int d) {
        for (int i = d; i < data.length; i++) {
            int j = i - d;
            int temp = data[i];
            while (j >= 0 && data[j] > temp) {
                data[j + d] = data[j];
                j -= d;
            }

            data[j + d] = temp;
        }
    }
}
```

## 归并排序

递归分治的思想，将一个数组不断二分，分到最小之后，排序，然后合起来。

时间复杂度$O(nlgn)$

空间复杂度$O(n)$

```java
public class MergeSort {
    public static void sort(int[] data) {
        mergeSort(data, 0, data.length - 1);
    }

    private static void mergeSort(int[] data, int left, int right) {
        if (left < right) {
            // 递归的思想
            int mid = (left + right) / 2;
            mergeSort(data, left, mid);
            mergeSort(data, mid + 1, right);
            merge(data, left, mid, right);
        }
    }

    private static void merge(int[] data, int left, int mid, int right) {
        int[] temp = new int[right - left + 1];

        int i = left;
        int j = mid + 1;
        int k = 0;

        // 直接选择然后复制过去
        while (i <= mid && j <= right) {
            if (data[i] <= data[j]) {
                temp[k ++] = data[i ++];
            }else {
                temp[k ++] = data[j ++];
            }
        }

        // 左边没有复制完的
        while (i <= mid) {
            temp[k ++] = data[i ++];
        }

        // 右边没有复制完的
        while (j <= right) {
            temp[k ++] = data[j ++];
        }

        // 将临时数组复制到真正的数组里面
        System.arraycopy(temp, 0, data, left, temp.length);
    }
}
```

## 计数排序

如果需要排序的数组的值的范围不是很大的时候，就可以通过下标计数的方法完成排序。

时间复杂度$O(n)$

空间复杂度 根据数组的值的范围而定

```java
public class CountSort {
    public static void sort(int[] data) {
        int[] a = min(data);
        int max = a[1];
        int min = a[0];
        int size = max - min;

        // 构建计数数组
        int[] count = new int[size + 1];
        Arrays.fill(count, 0);

        // 遍历一遍带排序数组，得到计数数组
        for (int d : data) {
            count[d - min]++;
        }

        // 对计数数组进行重整，得到排序后数组
        int k = 0;
        for (int i = 0; i <= size; i++) {
            for (int j = 0; j < count[i]; j++) {
                data[k ++] = i + min;
            }
        }
    }

    // 确定计数数组的范围
    private static int[] min(int[] data) {
        int min = data[0];
        int max = data[0];
        for (int i = 1; i < data.length; i++) {
            if (data[i] < min) {
                min = data[i];
            }else if (data[i] > max) {
                max = data[i];
            }
        }
        return new int[]{min, max};
    }
}
```

## 桶排序

桶排序的原理是，先将数组划分为多个桶，同时桶与桶之间是按序排列的大小关系，然后分别对每个桶中的数据进行排序，最后将桶中的数据还原到数组中。

这中间要明确两个概念，桶的数量 m 与 数组的大小 n

时间复杂度$O(n + n * (logn - logm))$

空间复杂度$O(n + m)$

```java
public class BucketSort {
    public static void sort(int[] data) {
        // 桶数量
        int bucketNum = 10;
        // 桶索引
        List<List<Integer>> buckets = new ArrayList<>();
        for (int i = 0; i < bucketNum; i++) {
            buckets.add(new LinkedList<>());
        }

        // 将数组的数据依次加入桶
        for (int aData : data) {
            buckets.get(f(aData)).add(aData);
        }

        // 对每个桶排序
        for (List<Integer> bucket : buckets) {
            if (!bucket.isEmpty()) {
                Collections.sort(bucket);
            }
        }

        // 将桶中的数据还原到数组中
        int k = 0;
        for (List<Integer> l : buckets) {
            for (int i : l) {
                data[k ++] = i;
            }
        }
     }

    private static int f(int x) {
        return x / 10;
    }
}
```

## 基数排序

基数排序比较有趣，他通过多对数组的遍历完成排序，按照数组中的最大值的位数，如 123， 3 位，就需要遍历 3 次，从这个数的最低位，开始，先完成数组中所有数据的个位数的排序（通过一个链表保存然后恢复，与桶排序类似原理），然后依次完成十位数、百位数排序，当最高位的排序完成的时候，因为之前的个位数与十位数已经是有序的了，那么一旦百位数的排序完成，那么这个数组就肯定也是一个有序的数组了。

时间复杂度$O(d(n + rd))$

空间复杂度$O(rd)$

```java
public class RadixSort {
    public static void sort(int[] data) {
        int max = getMaxBit(data);

        for (int i = 1; i <= max; i++) {
            List<List<Integer>> list = distribute(data, i);
            collect(data, list);
        }
    }

    // 对当前数组中按某一位进行排序
    private static List<List<Integer>> distribute(int[] data, int bit) {
        List<List<Integer>> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(new LinkedList<>());
        }

        for (int aData : data) {
            list.get(getBit(aData, bit)).add(aData);
        }
        return list;
    }

    // 将链表中保存的数据还原到数组
    private static void collect(int[] data, List<List<Integer>> buf) {
        int k = 0;
        for (List<Integer> l : buf) {
            for (int i : l) {
                data[k ++] = i;
            }
        }
    }

    // 获取数组中最大数的位数（以确定遍历次数）
    private static int getMaxBit(int[] data) {
        int num = 0;
        for (int i : data) {
            int l = (i + "").length();
            if (l > num) {
                num = l;
            }
        }
        return num;
    }

    // 获取某个数某位的值（没有则用 0 代替）
    private static int getBit(int x, int n) {
        String xs = x + "";
        if (xs.length() < n) {
            return 0;
        }else {
            return xs.charAt(xs.length() - n) - '0';
        }
    }
}
```

## 总结

### 比较

| 排序方法 |   平均时间   |   最坏情况   | 辅助存储  |
| :------: | :----------: | :----------: | :-------: |
| 简单排序 |   $O(n^2)$   |   $O(n^2)$   |  $O(1)$   |
| 快速排序 |  $O(nlgn)$   |   $O(n^2)$   | $O(logn)$ |
|  堆排序  |  $O(nlogn)$  |  $O(nlogn)$  |  $O(1)$   |
| 归并排序 |  $O(nlogn)$  |  $O(nlogn)$  |  $O(n)$   |
| 基数排序 | $O(d(n+rd))$ | $O(d(n+rd))$ |  $O(rd)$  |

### 稳定性

1. 稳定的
   - 基数排序
   - 冒泡排序
   - 插入排序
   - 归并排序
2. 不稳定的
   - 快速排序
   - 堆排序
   - 希尔排序
   - 选择排序