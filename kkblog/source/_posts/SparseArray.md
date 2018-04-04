---
title: SparseArray源码解析
date: 2017-09-07 20:31:14
tags: [SparseArray, HashMap, 二分查找, Android]
categories: [Android]
---

### 构造函数

```
    /**
     * Creates a new SparseArray containing no mappings that will not
     * require any additional memory allocation to store the specified
     * number of mappings.  If you supply an initial capacity of 0, the
     * sparse array will be initialized with a light-weight representation
     * not requiring any additional array allocations.
     */
    public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
```

* 查找

```
    public E get(int key) {
        return get(key, null);
    }

    /**
     * Gets the Object mapped from the specified key, or the specified Object
     * if no such mapping has been made.
     */
    @SuppressWarnings("unchecked")
    public E get(int key, E valueIfKeyNotFound) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i < 0 || mValues[i] == DELETED) {
            return valueIfKeyNotFound;
        } else {
            return (E) mValues[i];
        }
    }
```

* 插入

插入时，参数既要有key又要有value。
```
/**  put插入
     * Adds a mapping from the specified key to the specified value,
     * replacing the previous mapping from the specified key if there
     * was one.
     */
    public void put(int key, E value) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            mValues[i] = value;
        } else {
            i = ~i;
            // 空间已经有了,但是被删除了，那么重新赋值
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }
            // 空间超出，先gc一下
            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }
            // 扩展数组
            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++; // SparseArray大小增加
        }
    }
```

``` java
/**  append插入
     * Puts a key/value pair into the array, optimizing for the case where
     * the key is greater than all existing keys in the array.
     */
    public void append(int key, E value) {
        if (mSize != 0 && key <= mKeys[mSize - 1]) {
            put(key, value);
            return;
        }
        // 优化了key大于数组中所有key的情况，mKeys[mSize-1]是这个数组中最大的一个key
        if (mGarbage && mSize >= mKeys.length) {
            gc();
        }

        mKeys = GrowingArrayUtils.append(mKeys, mSize, key);
        mValues = GrowingArrayUtils.append(mValues, mSize, value);
        mSize++;
    }
```

* 整理空间
``` java
  private void gc() {
        // Log.e("SparseArray", "gc start with " + mSize);

        int n = mSize;
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;

        for (int i = 0; i < n; i++) {
            Object val = values[i];

            if (val != DELETED) {
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
                    values[i] = null;
                }

                o++;
            }
        }

        mGarbage = false;
        mSize = o;

        // Log.e("SparseArray", "gc end with " + mSize);
    }
```



* 删除
```
    /**
     * Removes the mapping from the specified key, if there was any.
     */
    public void delete(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            if (mValues[i] != DELETED) {
                mValues[i] = DELETED;
                mGarbage = true;
            }
        }
    }
```

* 最关键的二分法
```
    // This is Arrays.binarySearch(), but doesn't do any argument validation.
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
```

可以看出SparseArray里面的元素是按照key从小到大排序的，所以插入、删除、查找的速度都很快
