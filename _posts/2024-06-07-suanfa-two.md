---
title: C++ unordered_set 用法与例题
date: 2024-06-07 21:00:00 +0800
categories: [C++]
tags: [C++, STL, unordered_set, LeetCode]
---



### unordered_set

`unordered_set`与`set`非常类似,

唯一的区别是set会对存进去的数据进行排序，

而unordered_set是乱序排列。

`unordered_set`有如下三个**特性**：

1. 不再以键值对的形式存储数据，而是直接存储数据的值。而在关联式容器set中，是以键值对的方式存储的。且`set`与`map`又有所不同，set只能存储键与值相同的键值对，例如键为'a'，值为'a'。
2. 容器内部存储的元素的值各不相同，即天然去重。且不能被修改。注意：set也是天然去重。
3. 容器内的元素乱序存在。

### unordered_set成员方法

| 成员方法    | 功能                                                         |
| ----------- | ------------------------------------------------------------ |
| `find(key)` | 查找值为key的元素，如果找到，则返回一个指向该元素的正向迭代器；如果没找到，则返回一个与end()方法相同的迭代器 |
| `end()`     | 返回指向容器中最后一个元素之后位置的迭代器                   |

下面以力扣 349“两个数组的交集”为例：

> 给定两个整数数组 `nums1` 和 `nums2`，返回它们的交集。输出结果中的每个元素一定是唯一的，结果顺序可以任意。

[查看题目原文](https://leetcode.cn/problems/intersection-of-two-arrays/)

可以先把 `nums1` 放入 `unordered_set`，遍历 `nums2` 时查询元素是否存在，再用另一个集合保存结果以保证不重复：

```c++
class Solution {
public:
    vector<int> intersection(vector<int>& nums1, vector<int>& nums2) {
        unordered_set<int> result_set;
        unordered_set<int> nums_set(nums1.begin(),nums1.end());
        for(int num : nums2) {
            if(nums_set.find(num) != nums_set.end()) {
                result_set.insert(num);
            }
        }
        return vector<int>(result_set.begin(),result_set.end());
    }
};
```

### unordered_set的其他成员方法：

| 成员方法           | 功能                                                         |
| ------------------ | ------------------------------------------------------------ |
| begin()            | 返回指向容器中第一个元素的正向迭代器                         |
| end()              | 返回指向容器中最后一个元素之后位置的正向迭代器               |
| cbegin()           | 和 begin() 功能相同，只不过其返回的是 const 类型的正向迭代器 |
| cend()             | 和 end() 功能相同，只不过其返回的是 const 类型的正向迭代器   |
| empty()            | 若容器为空，则返回 true；否则 false                          |
| size()             | 返回当前容器中存有元素的个数                                 |
| max_size()         | 返回容器所能容纳元素的最大个数，不同的操作系统，其返回值亦不相同 |
| find(key)          | 查找以值为 key 的元素，如果找到，则返回一个指向该元素的正向迭代器；反之，则返回一个指向容器中最后一个元素之后位置的迭代器 |
| count(key)         | 在容器中查找值为 key 的元素的个数                            |
| equal_range(key)   | 返回一个 pair 对象，其包含 2 个迭代器，用于表明当前容器中值为 key 的元素所在的范围。 |
| emplace()          | 向容器中添加新元素，效率比 insert() 方法高。                 |
| emplace_hint()     | 向容器中添加新元素，效率比 insert() 方法高                   |
| insert()           | 向容器中添加新元素                                           |
| erase()            | 删除指定元素                                                 |
| clear()            | 清空容器，即删除容器中存储的所有元素                         |
| swap()             | 交换 2 个 unordered_map 容器存储的元素，前提是必须保证这 2 个容器的类型完全相等 |
| bucket_count()     | 返回当前容器底层存储元素时，使用桶（一个线性链表代表一个桶）的数量 |
| max_bucket_count() | 返回当前系统中，unordered_map 容器底层最多可以使用多少桶     |
| bucket_size()      | 返回第 n 个桶中存储元素的数量                                |
| bucket()           | 返回值为 key 的元素所在桶的编号                              |

