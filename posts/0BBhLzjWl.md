---
title: '数据结构常见的八大排序算法'
date: 2021-04-25 16:06:18
tags: [java,算法]
published: true
hideInList: false
feature: /post-images/0BBhLzjWl.png
isTop: false
---
八大排序，三大查找是《数据结构》当中非常基础的知识点，在这里为了复习顺带总结了一下常见的八种排序算法。

常见的八大排序算法，他们之间关系如下：

![](https://tinaxiawuhao.github.io/post-images/1619339616659.png)

 

他们的性能比较：

![](https://tinaxiawuhao.github.io/post-images/1619339624226.png)

下面，利用java分别将他们进行实现。

## 直接插入排序

### 算法思想：

 

![](https://tinaxiawuhao.github.io/post-images/1619339633556.gif)

 

直接插入排序的核心思想就是：将数组中的所有元素依次跟前面已经排好的元素相比较，如果选择的元素比已排序的元素小，则交换，直到全部元素都比较过。

因此，从上面的描述中我们可以发现，直接插入排序可以用两个循环完成：

1. 第一层循环：遍历待比较的所有数组元素
2. 第二层循环：将本轮选择的元素(selected)与已经排好序的元素(ordered)相比较。

如果：selected < ordered，那么将二者交换

### 代码实现

```java
package sort;

import java.util.Arrays;

/**
 * @author wuhao
 * @desc 直接插入排序
 * @date 2020-12-02 15:04:57
 * 直接插入排序的核心思想就是：将数组中的所有元素依次跟前面已经排好的元素相比较，如果选择的元素比已排序的元素小，则交换，直到全部元素都比较过。
 * 因此，从上面的描述中我们可以发现，直接插入排序可以用两个循环完成：
 * 第一层循环：遍历待比较的所有数组元素
 * 第二层循环：将本轮选择的元素(selected)与已经排好序的元素(ordered)相比较。
 * 如果：selected < ordered，那么将二者交换
 */
public class DirectInsertionSort {
    public static void main(String[] args) {
        int[] arr = {12, 32, 22, 7, 48};
        insertSort(arr);
    }

    private static void insertSort(int[] arr) {
       for (int i = 1; i < arr.length; i++) {
           int temp = arr[i];
           int j;
           for (j = i - 1; j >= 0; j--) {
               if (temp < arr[j]) {
                   arr[j + 1] = arr[j];
               } else {
                   break;
               }
           }
           arr[j + 1] = temp;
       }

        System.out.println(Arrays.toString(arr));
    }
}

```

## 希尔排序

### 算法思想：

 

![](https://tinaxiawuhao.github.io/post-images/1619339644637.png)
 

希尔排序的算法思想：将待排序数组按照步长gap进行分组，然后将每组的元素利用直接插入排序的方法进行排序；每次将gap折半减小，循环上述操作；当gap=1时，利用直接插入，完成排序。

同样的：从上面的描述中我们可以发现：希尔排序的总体实现应该由三个循环完成：

1. 第一层循环：将gap依次折半，对序列进行分组，直到gap=1
2. 第二、三层循环：也即直接插入排序所需要的两次循环。具体描述见上。

### 代码实现

```java
package sort;

import java.util.Arrays;

/**
 * @author wuhao
 * @desc 希尔排序
 * @date 2020-12-02 15:20:43
 * 希尔排序的算法思想：将待排序数组按照步长gap进行分组，然后将每组的元素利用直接插入排序的方法进行排序；每次将gap折半减小，循环上述操作；当gap=1时，利用直接插入，完成排序。
 * 同样的：从上面的描述中我们可以发现：希尔排序的总体实现应该由三个循环完成：
 * 第一层循环：将gap依次折半，对序列进行分组，直到gap=1
 * 第二、三层循环：也即直接插入排序所需要的两次循环。具体描述见上。
 */
public class HillSort {
    public static void main(String[] args) {
        int[] arr = {12, 32, 22, 7, 48, 3, 5, 6, 8, 24};
        shellSort(arr);
    }

    private static void shellSort(int[] arr) {
        for (int step = arr.length / 2; step > 0; step /= 2) {
//            for (int i = step; i < arr.length; i++) {
//                int temp = arr[i];
//                int j;
//                for (j = i - step; j >= 0 && arr[j] > temp; j -= step) {
//                    arr[j + step] = arr[j];
//                }
//                arr[j + step] = temp;
//            }
            for (int i = step; i < arr.length; i++) {
                int k=i;
                for (int j = i-step; j >=0 && arr[j] > arr[k]; j -= step,k -=step) {
                   int temp = arr[j];
                    arr[j]= arr[k];
                    arr[k]=temp;
                }
            }
        }

        System.out.println(Arrays.toString(arr));

    }
}

```



## 简单选择排序

### 算法思想

 

![](https://tinaxiawuhao.github.io/post-images/1619339776440.gif)

 

简单选择排序的基本思想：比较+交换。

1. 从待排序序列中，找到关键字最小的元素；
2. 如果最小元素不是待排序序列的第一个元素，将其和第一个元素互换；
3. 从余下的 N - 1 个元素中，找出关键字最小的元素，重复(1)、(2)步，直到排序结束。

因此我们可以发现，简单选择排序也是通过两层循环实现。

第一层循环：依次遍历序列当中的每一个元素

第二层循环：将遍历得到的当前元素依次与余下的元素进行比较，符合最小元素的条件，则交换。

### 代码实现

```java
package sort;

import java.util.Arrays;

/**
 * @author wuhao
 * @desc 简单选择排序
 * @date 2020-12-02 16:21:47
 * 简单选择排序的基本思想：比较+交换。
 * 从待排序序列中，找到关键字最小的元素；
 * 如果最小元素不是待排序序列的第一个元素，将其和第一个元素互换；
 * 从余下的 N - 1 个元素中，找出关键字最小的元素，重复(1)、(2)步，直到排序结束。
 * 因此我们可以发现，简单选择排序也是通过两层循环实现。
 * 第一层循环：依次遍历序列当中的每一个元素
 * 第二层循环：将遍历得到的当前元素依次与余下的元素进行比较，符合最小元素的条件，则交换。
 */
public class SimpleSelectionSort {
    public static void main(String[] args) {
        int[] arr = {12, 32, 22, 7, 48, 3, 5, 6, 8, 24};
        selectionSort(arr);
    }

    private static void selectionSort(int[] arr) {
        for (int i = 0; i < arr.length; i++) {
            int k = i;
            for (int j = i + 1; j < arr.length; j++) {
                if (arr[j] < arr[k]) {
                    k = j;
                }
            }
            if (k != i) {
                int temp = arr[i];
                arr[i] = arr[k];
                arr[k] = temp;
            }


        }
        System.out.println(Arrays.toString(arr));
    }
}
```



## 堆排序

### 堆的概念

堆：本质是一种数组对象。特别重要的一点性质：<b>任意的叶子节点小于（或大于）它所有的父节点</b>。对此，又分为大顶堆和小顶堆，大顶堆要求节点的元素都要大于其孩子，小顶堆要求节点元素都小于其左右孩子，两者对左右孩子的大小关系不做任何要求。

利用堆排序，就是基于大顶堆或者小顶堆的一种排序方法。下面，我们通过大顶堆来实现。

### 基本思想：

堆排序可以按照以下步骤来完成：

![](https://tinaxiawuhao.github.io/post-images/1619339787440.png)

构建大顶堆.png

 

![](https://tinaxiawuhao.github.io/post-images/1619339794481.png)

Paste_Image.png

1. 构建初始堆，将待排序列构成一个大顶堆(或者小顶堆)，升序大顶堆，降序小顶堆；
2. 将堆顶元素与堆尾元素交换，并断开(从待排序列中移除)堆尾元素。
3. 重新构建堆。
4. 重复2~3，直到待排序列中只剩下一个元素(堆顶元素)。

### 代码实现：

```java
package sort;

import java.util.Arrays;

/**
 * @author wuhao
 * @desc 堆排序
 * @date 2020-12-04 09:13:58
 * 堆排序可以按照以下步骤来完成：
 * 首先将序列构建称为大顶堆；
 * （这样满足了大顶堆那条性质：位于根节点的元素一定是当前序列的最大值）
 * 取出当前大顶堆的根节点，将其与序列末尾元素进行交换；
 * （此时：序列末尾的元素为已排序的最大值；由于交换了元素，当前位于根节点的堆并不一定满足大顶堆的性质）
 * 对交换后的n-1个序列元素进行调整，使其满足大顶堆的性质；
 * 重复2.3步骤，直至堆中只有1个元素为止
 */
public class HeapSort {
    public static void main(String[] args) {
        int[] arr = {11, 44, 23, 67, 88, 65, 34, 48, 9, 12};
        heapSort(arr);
        System.out.println(Arrays.toString(arr));
    }

    private static void heapSort(int[] a) {
        // 首先需要创建根堆
        for (int i = a.length / 2; i >= 0; i--) { // 从最后一个非终端结点开始，然后一次--
            HeapAdjust(a, i, a.length);
        }

        for (int i = a.length - 1; i > 0; --i) {// 这个循环是把最大值a[0]放到末尾 ，
            int temp = a[0];
            a[0] = a[i]; // 此时i代表最后一个元素
            a[i] = temp;
            HeapAdjust(a, 0, i );
        }
    }

    // 调整堆
    private static void HeapAdjust(int[] a, int parent, int m) {// parent代表当前 m代表最后
        int temp = a[parent]; // 先把a[parent]的值赋给temp保存起来
        for (int j = 2 * parent; j < m; j *= 2) {
            if (j+1 < m && a[j] < a[j + 1]) { // 判断是a[parent]大还是a[j + 1]大，如果a[j + 1]大 就++j，把j换成当前最大
                j++;
            }
            if (temp >= a[j]) { // 如果temp中比最大值还大，代表本身就是一个根堆，break
                break;// 如果大于，就代表当前为大跟对，退出
            }
            a[parent] = a[j];// 否则就把最大给[parent]
            parent = j;// 然后把最大下标给parent，继续循环,检查是否因为调整根堆而破坏了子树
        }
        a[parent] = temp;
    }

    /**
     * 创建堆，
     * @param arr 待排序列
     */
//    private static void heapSort(int[] arr) {
//        //创建堆
//        for (int i = (arr.length - 1) / 2; i >= 0; i--) {
//            //从第一个非叶子结点从下至上，从右至左调整结构
//            adjustHeap(arr, i, arr.length);
//        }
//
//        //调整堆结构+交换堆顶元素与末尾元素
//        for (int i = arr.length - 1; i > 0; i--) {
//            //将堆顶元素与末尾元素进行交换
//            int temp = arr[i];
//            arr[i] = arr[0];
//            arr[0] = temp;
//
//            //重新对堆进行调整
//            adjustHeap(arr, 0, i);
//        }
//    }

    /**
     * 调整堆
     * @param arr 待排序列
     * @param parent 父节点
     * @param length 待排序列尾元素索引
     */
//    private static void adjustHeap(int[] arr, int parent, int length) {
//        //将temp作为父节点
//        int temp = arr[parent];
//        //左孩子
//        int lChild = 2 * parent + 1;
//
//        while (lChild < length) {
//            //右孩子
//            int rChild = lChild + 1;
//            // 如果有右孩子结点，并且右孩子结点的值大于左孩子结点，则选取右孩子结点
//            if (rChild < length && arr[lChild] < arr[rChild]) {
//                lChild++;
//            }
//
//            // 如果父结点的值已经大于孩子结点的值，则直接结束
//            if (temp >= arr[lChild]) {
//                break;
//            }
//
//            // 把孩子结点的值赋给父结点
//            arr[parent] = arr[lChild];
//
//            //选取孩子结点的左孩子结点,继续向下筛选
//            parent = lChild;
//            lChild = 2 * lChild + 1;
//        }
//        arr[parent] = temp;
//    }

}

```



## 冒泡排序

### 基本思想

 

![](https://tinaxiawuhao.github.io/post-images/1619339811436.gif)



冒泡排序思路比较简单：

- 1. 将序列当中的左右元素，依次比较，保证右边的元素始终大于左边的元素；

（ 第一轮结束后，序列最后一个元素一定是当前序列的最大值；）

- 1. 对序列当中剩下的n-1个元素再次执行步骤1。
  1. 对于长度为n的序列，一共需要执行n-1轮比较

（利用while循环可以减少执行次数）

### 代码实现

```java
package sort;

import java.util.Arrays;

/**
 * @author wuhao
 * @desc 冒泡排序
 * @date 2020-12-02 16:34:57
 * 冒泡排序思路比较简单：
 * 将序列当中的左右元素，依次比较，保证右边的元素始终大于左边的元素；
 * （ 第一轮结束后，序列最后一个元素一定是当前序列的最大值；）
 * 对序列当中剩下的n-1个元素再次执行步骤1。
 * 对于长度为n的序列，一共需要执行n-1轮比较
 * （利用while循环可以减少执行次数）
 */
public class BubbleSort {
    public static void main(String[] args) {
        int[] arr = {12, 32, 22, 7, 48};
        bubbleSort(arr);
    }

    private static void bubbleSort(int[] arr) {
        for (int i = 0; i < arr.length; i++) {
            for (int j = 0; j < i; j++) {
                if (arr[i] < arr[j]) {
                    int temp=arr[j];
                    arr[j]=arr[i];
                    arr[i]=temp;
                }
            }
        }
        System.out.println(Arrays.toString(arr));
    }
}

```



## 快速排序

### 算法思想：

 

![](https://tinaxiawuhao.github.io/post-images/1619339820307.gif)

快速排序的基本思想：挖坑填数+分治法

- 1. 从序列当中选择一个基准数(pivot)

在这里我们选择序列当中第一个数为基准数

- 1. 将序列当中的所有数依次遍历，比基准数大的位于其右侧，比基准数小的位于其左侧
  1. 重复步骤a.b，直到所有子集当中只有一个元素为止。

用伪代码描述如下：

1．i =L; j = R; 将基准数挖出形成第一个坑a[i]。

2．j--由后向前找比它小的数，找到后挖出此数填前一个坑a[i]中。

3．i++由前向后找比它大的数，找到后也挖出此数填到前一个坑a[j]中。

4．再重复执行2，3二步，直到i==j，将基准数填入a[i]中

### 代码实现：

```java
package sort;

import java.util.Arrays;
import java.util.Stack;

/**
 * @author wuhao
 * @desc 快速排序
 * @date 2020-12-02 17:08:57
 * 快速排序的基本思想：挖坑填数+分治法
 * 从序列当中选择一个基准数(pivot)
 * 在这里我们选择序列当中第一个数为基准数
 * 将序列当中的所有数依次遍历，比基准数大的位于其右侧，比基准数小的位于其左侧
 * 重复步骤a.b，直到所有子集当中只有一个元素为止。
 * 用伪代码描述如下：
 * 1．i =L; j = R; 将基准数挖出形成第一个坑a[i]。
 * 2．j--由后向前找比它小的数，找到后挖出此数填前一个坑a[i]中。
 * 3．i++由前向后找比它大的数，找到后也挖出此数填到前一个坑a[j]中。
 * 4．再重复执行2，3二步，直到i==j，将基准数填入a[i]中
 */
public class QuickSort {
    public static void main(String[] args) {
        int[] arr = {12, 32, 22, 7, 48, 3, 35, 6, 8, 42};
//        quickSort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr));

        /*-----------非递归实现----------*/
        sort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr));
    }

    private static void quickSort(int[] arr, int left, int right) {
        if (left > right) {
            return;
        }
        // base中存放基准数
        int base = arr[left];
        int i = left, j = right;
        while (i != j) {
            // 顺序很重要，先从右边开始往左找，直到找到比base值小的数
            while (arr[j] >= base && i < j) {
                j--;
            }

            // 再从左往右边找，直到找到比base值大的数
            while (arr[i] <= base && i < j) {
                i++;
            }

            // 上面的循环结束表示找到了位置或者(i>=j)了，交换两个数在数组中的位置
            if (i < j) {
                int tmp = arr[i];
                arr[i] = arr[j];
                arr[j] = tmp;
            }
        }
        // 将基准数放到中间的位置（基准数归位）
        arr[left] = arr[i];
        arr[i] = base;

        // 递归，继续向基准的左右两边执行和上面同样的操作
        // i的索引处为上面已确定好的基准值的位置，无需再处理
        quickSort(arr, left, i - 1);
        quickSort(arr, i + 1, right);

    }


    /*-----------------------------非递归实现------------------------------*/
    public static void sort(int[] arr, int left, int right) {
        int privot, top, last;
        Stack<Integer> s = new Stack<>();
        privot = QuickSort(arr, left, right);
        if (privot > left + 1) {
            s.push(left);
            s.push(privot - 1);

        }
        if (privot < right - 1) {
            s.push(privot + 1);
            s.push(right);
        }
        while (!s.empty()) {
            top = s.pop();
            last = s.pop();
            privot = QuickSort(arr, last, top);
            if (privot > last + 1) {
                s.push(last);
                s.push(privot - 1);
            }
            if (privot < top - 1) {
                //System.out.println(top);
                s.push(privot + 1);
                s.push(top);
            }
        }
    }

    public static int QuickSort(int[] arr, int left, int right) {
        int privot = left;
        while (left < right) {
            while ((arr[privot] < arr[right]) & left < right) {
                right--;
            }
            int temp = arr[privot];
            arr[privot] = arr[right];
            arr[right] = temp;
            privot = right;

            while ((arr[privot] > arr[left]) & left < right) {
                left++;
            }
            temp = arr[privot];
            arr[privot] = arr[left];
            arr[left] = temp;
            privot = left;
        }
        return privot;
    }
}

```



## 归并排序

### 算法思想：

![](https://tinaxiawuhao.github.io/post-images/1619339833766.png)


- 1. 归并排序是建立在归并操作上的一种有效的排序算法，该算法是采用0的一个典型的应用。它的基本操作是：将已有的子序列合并，达到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。
  1. 归并排序其实要做两件事：

- - 分解----将序列每次折半拆分
  - 合并----将划分后的序列段两两排序合并

因此，归并排序实际上就是两个操作，拆分+合并

- 1. 如何合并？

L[first...mid]为第一段，L[mid+1...last]为第二段，并且两端已经有序，现在我们要将两端合成达到L[first...last]并且也有序。

- - 首先依次从第一段与第二段中取出元素比较，将较小的元素赋值给temp[]
  - 重复执行上一步，当某一段赋值结束，则将另一段剩下的元素赋值给temp[]
  - 此时将temp[]中的元素复制给L[]，则得到的L[first...last]有序

- 1. 如何分解？

在这里，我们采用递归的方法，首先将待排序列分成A,B两组；然后重复对A、B序列

分组；直到分组后组内只有一个元素，此时我们认为组内所有元素有序，则分组结束。

### 代码实现

```java
package sort;

import java.util.Arrays;

/**
 * @author wuhao
 * @desc 归并排序
 * @date 2020-12-03 09:35:00
 * 归并排序是建立在归并操作上的一种有效的排序算法，该算法是采用0的一个典型的应用。它的基本操作是：将已有的子序列合并，达到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。
 * 归并排序其实要做两件事：
 * 分解----将序列每次折半拆分
 * 合并----将划分后的序列段两两排序合并
 * 因此，归并排序实际上就是两个操作，拆分+合并
 * 如何合并？
 * L[first...mid]为第一段，L[mid+1...last]为第二段，并且两端已经有序，现在我们要将两端合成达到L[first...last]并且也有序。
 * 首先依次从第一段与第二段中取出元素比较，将较小的元素赋值给temp[]
 * 重复执行上一步，当某一段赋值结束，则将另一段剩下的元素赋值给temp[]
 * 此时将temp[]中的元素复制给L[]，则得到的L[first...last]有序
 * 如何分解？
 * 在这里，我们采用递归的方法，首先将待排序列分成A,B两组；然后重复对A、B序列
 * 分组；直到分组后组内只有一个元素，此时我们认为组内所有元素有序，则分组结束。
 */
public class MergeSort {
    public static void main(String[] args) {
        int[] arr = {11, 44, 23, 67, 88, 65, 34, 48, 9, 12};
        int[] tmp = new int[arr.length];    //新建一个临时数组存放
        mergeSort(arr, 0, arr.length - 1, tmp);
        System.out.println(Arrays.toString(arr));
    }

    private static void mergeSort(int[] arr, int low, int high, int[] tmp) {
        if (low < high) {
            int mid = (low + high) / 2;
            mergeSort(arr, low, mid, tmp);
            mergeSort(arr, mid + 1, high, tmp);
            merge(arr, low, mid, high, tmp);
        }
    }

    private static void merge(int[] arr, int low, int mid, int high, int[] tmp) {
        int i = 0;
        int j = low, k = mid + 1;
        while (j <= mid && k <= high) {
            if (arr[j] < arr[k]) {
                tmp[i++] = arr[j++];
            } else {
                tmp[i++] = arr[k++];
            }
        }
        while (j <= mid) {
            tmp[i++] = arr[j++];
        }
        while (k <= high) {
            tmp[i++] = arr[k++];
        }
        for (int l = 0; l < i; l++) {
            arr[low+l] = tmp[l];
        }

    }
}

```



## 基数排序

### 算法思想

 
![](https://tinaxiawuhao.github.io/post-images/1619339840696.gif)


- 1. 基数排序：通过序列中各个元素的值，对排序的N个元素进行若干趟的“分配”与“收集”来实现排序。

分配：我们将L[i]中的元素取出，首先确定其个位上的数字，根据该数字分配到与之序号相同的桶中

收集：当序列中所有的元素都分配到对应的桶中，再按照顺序依次将桶中的元素收集形成新的一个待排序列L[ ]

对新形成的序列L[]重复执行分配和收集元素中的十位、百位...直到分配完该序列中的最高位，则排序结束

### 代码实现

```java
package sort;

import java.util.Arrays;

/**
 * @author wuhao
 * @desc 基数排序
 * @date 2020-12-03 09:58:32
 * 基数排序：通过序列中各个元素的值，对排序的N个元素进行若干趟的“分配”与“收集”来实现排序。
 * 分配：我们将L[i]中的元素取出，首先确定其个位上的数字，根据该数字分配到与之序号相同的桶中
 * 收集：当序列中所有的元素都分配到对应的桶中，再按照顺序依次将桶中的元素收集形成新的一个待排序列L[ ]
 * 对新形成的序列L[]重复执行分配和收集元素中的十位、百位...直到分配完该序列中的最高位，则排序结束
 * 根据上述“基数排序”的展示，我们可以清楚的看到整个实现的过程
 */
public class BaseSort {
    public static void main(String[] args) {
        int[] arr = {63, 157, 189, 51, 101, 47, 141, 121, 157, 156,
                194, 117, 98, 139, 67, 133, 181, 12, 28, 0, 109};

        radixSort(arr);
        System.out.println(Arrays.toString(arr));
    }

    private static void radixSort(int[] arr) {
        //待排序列最大值
        int max = arr[0];
        int exp;//指数
        //计算最大值
        for (int anArr : arr) {
            if (anArr > max) {
                max = anArr;
            }
        }
        //从个位开始，对数组进行排序
        for (exp = 1; max / exp > 0; exp *= 10) {
            //存储待排元素的临时数组
            int[] temp = new int[arr.length];
            //分桶个数
            int[] buckets = new int[10];

            //将数据出现的次数存储在buckets中
            for (int value : arr) {
                //(value / exp) % 10 :value的最底位(个位)
                buckets[(value / exp) % 10]++;
            }

            //更改buckets[i]，
            for (int i = 1; i < 10; i++) {
                buckets[i] += buckets[i - 1];
            }

            //将数据存储到临时数组temp中
            for (int i = arr.length - 1; i >= 0; i--) {
                temp[buckets[(arr[i] / exp) % 10] - 1] = arr[i];
                buckets[(arr[i] / exp) % 10]--;
            }

            //将有序元素temp赋给arr
            System.arraycopy(temp, 0, arr, 0, arr.length);
        }

    }
}
```



## 后记

写完之后运行了一下时间比较：

从运行结果上来看，堆排序、归并排序、基数排序是真的快。

对于快速排序迭代深度超过的问题，可以将考虑将快排通过非递归的方式进行实现。