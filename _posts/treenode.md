---
title: 二叉树
date: 2019-01-15 13:52:53
categories:
- 笔记
tags:
- Python3
---

`LeetCode` 二叉树题解汇总

<!-- more -->

<!-- toc -->

## 二叉树定义

```python
class TreeNode:
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None
```

## 相同的树

```python
def is_same_tree(p, q):
      if p is None:
          return not q
      if q is None:
          return not p
      return p.val == q.val and is_same_tree(p.left, q.left) and is_same_tree(p.right, q.right)

```

## 对称的树

```python
def is_symmetric(root):
    if not root:
        return True
    return symmetric(root.left, root.right)


def symmetric(l1, l2):
    if l1 is None:
        return not l2
    if l2 is None:
        return not l1
    return l1.val == l2.val and symmetric(l1.left, l2.right) and symmetric(l1.right, l2.left)

```

## 层次遍历

给定一个二叉树，返回其按层次遍历的节点值。

```python
def add(node, level, res):
    if node is None:
        return
    if len(res) < level:
        res.append([])
    res[level - 1].append(node.val)
    add(node.left, level + 1, res)
    add(node.right, level + 1, res)

def level_order(root):
    res = []
    add(root, 1, res)
    return res

```

## 最大深度

给定一个二叉树，找出其最大深度。

二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。

```python
def max_depth(root):
    return 1 + max(map(max_depth, (root.left, root.right))) if root else 0

```

## 最小深度

给定一个二叉树，找出其最小深度。

最小深度是从根节点到最近叶子节点的最短路径上的节点数量。

```python
def min_depth(root):
    if root is None:
        return 0
    if root.left is None:
        return 1 + min_depth(root.right)
    if root.right is None:
        return 1 + min_depth(root.left)
    return 1 + min(map(min_depth, (root.left, root.right)))

# 更简洁的写法
def min_depth(root):
    if root is None:
        return 0
    depth_under_root = map(min_depth, (root.left, root.right))
    return 1 + (min(depth_under_root) or max(depth_under_root))

```

## 将有序数组转化为二叉树

将一个按照升序排列的有序数组，转换为一棵高度平衡二叉搜索树。

高度平衡二叉树是指一个二叉树每个节点的左右两个子树的高度差的绝对值不超过1。

```python
def sorted_array_to_balanced_tree(nums):
    if not nums:
        return None
    mid = len(nums) // 2
    root = TreeNode(nums[mid])
    root.left = sorted_array_to_balanced_tree(nums[:mid])
    root.right = sorted_array_to_balanced_tree(nums[mid + 1:])

```

## 平衡二叉树

一个二叉树每个节点的左右两个子树的高度差的绝对值不超过1。

```python
def hight(node):
    if node is None:
        return 0
    return 1 + max(map(hight, (node.left, node.right)))

def is_balanced(root):
    if root is None:
        return True
    return abs(hight(root.left) - hight(root.right)) <= 1 and is_balanced(root.left) and is_balanced(root.right)

```

## 路径总和

给定一个二叉树和一个目标和，判断该树中是否存在根节点到叶子节点的路径，这条路径上所有节点值相加等于目标和。

```python
def has_path_sum(root, sums):
    if root is None:
        return False
    if root.left or root.right:
        return has_path_sum(root.left, sums - root.val) or has_path_sum(root.right, sums - root.val)
    return sums == root.val

```
