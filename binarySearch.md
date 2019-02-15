# 二叉搜索

二叉搜索一定要有偏向,否则会有边界搜索不到.

## 左偏向

```c++
while (left < right) {
            int mid = (left + right) / 2;
            if (nums[mid] < target) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
```

1. 如果左偏向,left值一定要在每次都更新,否则会死循环

## 右偏向

```c++
while (left < right) {
            int mid = (left + right) / 2 + 1;
            if (nums[mid] > target) {
                right = mid - 1;
            } else {
                left = mid;
            }
        }
```

1. 如果右偏向,right值一定要在每次都更新,否则会死循环