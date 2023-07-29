---
layout:     post
title:      "LeetCode中BackTracking相关算法题整理"
subtitle:   "BackTracking in LeetCode"
date:       2019-10-08
author:     LiuShuo
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Algorithm
---
    
> 关于回溯(backtracking)在LeetCode中出现的题目的相关解法的总结。注：所有代码来自互联网。

### Subset(78)
#### 思路一
> 每次递归都认为是最新的结果，将子结果集加入到结果集列表中，然后对剩下的位置的数据进行迭代+递归的处理方式。
>
> 默认将当前元素加入到子集合中，然后递归处理后面的数据，处理完毕后回到当前上下文，再将当前位置的数据从子集合中删除，继续从下一个位置继续处理。
>
> 注意迭代和递归的分工不同：
>
> 迭代是每一次都排除了之前位置的元素，只包括本次迭代位置的元素；
>
> 而递归处理是在迭代内部的逻辑，是在包括本次迭代位置元素的同时继续递归处理后面的元素。
>
> 该方案在递归层层深入时，边深入边收集结果，属于饥渴型，这里称之为饿汉法。

```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> list = new ArrayList<>();
    backtrack(list, new ArrayList<>(), nums, 0);
    return list;
}

private void backtrack(List<List<Integer>> list , List<Integer> tempList, int [] nums, int start){
    // 每次产生一个新的list，就直接加到结果里（饥渴型）
    list.add(new ArrayList<>(tempList)); // use new
    // 注意start要作为一个循环的开头，每次start加一，每个start开始都是一组set，里面包含当前的start，和后面出现或者不出现的每个数字
    // 每一个循环开始时都不包括之前的位置的元素，只从当前位置开始
    for(int i = start; i < nums.length; i++){
        // 每一个循环内都将本次循环的位置加入到子集合中
        tempList.add(nums[i]);
        // 从i+1开始继续递归，注意tempList必须包含本次迭代位置的元素
        backtrack(list, tempList, nums, i + 1); 
        // 每一个迭代结束前都将本次位置的元素从子集合中删除
        tempList.remove(tempList.size() - 1);
    }
}
```
#### 思路二
> 数组中从0开始到nums.length个数，每个位置的数在某一轮要么在结果集里，要么不在，所以每一个位置可以有两种处理case。
>
> 所以pos会一直递进，一直到数组尾部，而在递进的过程中每一轮都有两种情况，通过递归可以实现各种情况的组合，只需要在处理到末尾的时候将结果保存即可。
>
> 该方案在递归层层深入时，只有触达了最底层才会收集结果，属于懒汉型。
 


```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    backtrack(res, nums, 0, new ArrayList<>());
    return res;
}

private void backtrack(List<List<Integer>> res, int[] nums, int pos, List<Integer> subset) {
    if (pos == nums.length) { // pos已经达到了数组的最大下标，保存当前的subset（懒汉型）
        res.add(new ArrayList<>(subset)); // use new
        return;
    }
    backtrack(res, nums, pos + 1, subset); // Without current number.
    subset.add(nums[pos]);
    backtrack(res, nums, pos + 1, subset); // With current number.
    subset.remove(subset.size() - 1); // Reset.
}
```
### Subset(90)
> 思路和78题的思路一一样，因为需要去重，所以先排序后处理起来比较方便，而前后相邻元素要进行比对，所以使用 `for` 循环的方式引入下标会比较方便。

```java
public List<List<Integer>> subsetsWithDup(int[] nums) {
    List<List<Integer>> list = new ArrayList<>();
    // 因为存在duplicates所以关键的一步就是sorting
    Arrays.sort(nums);
    backtrack(list, new ArrayList<>(), nums, 0);
    return list;
}

private void backtrack(List<List<Integer>> list, List<Integer> tempList, int[] nums, int start){
    // 每次产生一个新的list，就直接加到结果里（饥渴型）
    list.add(new ArrayList<>(tempList)); // use new
    for(int i = start; i < nums.length; i++){
        if(i > start && nums[i] == nums[i-1]) 
            continue; // skip if duplicates with previous
            
        tempList.add(nums[i]);
        backtrack(list, tempList, nums, i + 1);
        tempList.remove(tempList.size() - 1);
    }
} 
```
### Combination Sum (39)
```java
public List<List<Integer>> combinationSum(int[] candidates, int target) {
    if (candidates == null || candidates.length == 0) {
        return Collections.emptyList();
    }
    final List<List<Integer>> res = new ArrayList<>(); // final
    Arrays.sort(candidates);
    helper(candidates, target, 0, new ArrayList<>(), res);
    return res;
}

private void backtrack(int[] candidates, int target, int pos, List<Integer> comb, List<List<Integer>> res) {
    if (target == 0) {
        res.add(new ArrayList<>(comb)); // use new
        return;
    }
    for (int i = pos; i < candidates.length; i++) {
        int newTarget = target - candidates[i]; // remaining after current counted
        if (newTarget >= 0) { // still not the end
            comb.add(candidates[i]);
            // Note i, numbers can be used unlimited number of times
            backtrack(candidates, newTarget, i, comb, res); 
            comb.remove(comb.size() - 1);
        } else {
            break; // Too big
        }
    }
}
```

### Combination Sum 2(40)
```java
public List<List<Integer>> dfs(int[] num, int target) {
    if (num == null || num.length == 0) {
        return Collections.emptyList();
    }
    final List<List<Integer>> res = new ArrayList<>(); // final
    Arrays.sort(num); // sort asc
    dfs(num, target, 0, new ArrayList<>(), res);
    return res;
}

/**
 * Skip duplicates after new target is generated
 */
public void dfs(int[] num, int target, int index, List<Integer> comb, List<List<Integer>> result) {
    if (target == 0) {
        result.add(new ArrayList<>(comb)); // use new
        return;
    }
    
    for (int i = index; i < num.length; i++) {
        int newTarget = target - num[i];
        if (newTarget >= 0) {
            comb.add(num[i]);
            dfs(num, newTarget, i + 1, comb, result);
            comb.remove(comb.size() - 1); // remove the last
        } else {
            break; // Too big
        }
        
        // Skip duplicates
        while (i < num.length - 1 && num[i] == num[i + 1]) {
            i++; // Move i to the end of the duplicates
        }
        // i++ of the for loop will move i to next number that is not a duplicate
    }
}
```
### Permutations(46)
#### 思路一
> 使用迭代+递归：迭代全部范围的元素，在内部进行判重，不重复，则递归前将本元素加入子集合中，回归后将元素删除，继续迭代下一个元素。 

```java
public List<List<Integer>> permute(int[] nums) {
   List<List<Integer>> list = new ArrayList<>();
   // Arrays.sort(nums); // not necessary
   // 因为[顺序重要] [元素的数量不重要]，所以我们就不使用start变量去控制有多少元素存在于templist中
   backtrack(list, new ArrayList<>(), nums);
   return list;
}

private void backtrack(List<List<Integer>> list, List<Integer> tempList, int [] nums){
   if(tempList.size() == nums.length){ // 懒汉型，只有在长度满足数组长度时才记录结果
      list.add(new ArrayList<>(tempList)); // use new
   } else{
      for(int i = 0; i < nums.length; i++){ 
         if(tempList.contains(nums[i])) 
             continue; // element already exists, skip
         tempList.add(nums[i]);
         backtrack(list, tempList, nums);
         tempList.remove(tempList.size() - 1);
      }
   }
} 

```
### Permutations 2(46)
```java
public List<List<Integer>> permuteUnique(int[] nums) {
    final List<List<Integer>> list = new ArrayList<>();
    Arrays.sort(nums);
    backtrack(list, new ArrayList<>(), nums, new boolean[nums.length]);
    return list;
}

private void backtrack(List<List<Integer>> list, List<Integer> tempList, int [] nums, boolean [] used){
    if(tempList.size() == nums.length){
        list.add(new ArrayList<>(tempList));
    } else{
        for(int i = 0; i < nums.length; i++){
            // 如果当前位置已经被使用，或者它和前面的位置的元素相同且前面位置的元素未被使用则继续循环
            // 前面的未被使用的原因就是发生了冲突，否则会跳过if设置对应的used[i]
            if(used[i] || i > 0 && nums[i] == nums[i-1] && !used[i - 1]) continue;
            used[i] = true; 
            tempList.add(nums[i]);
            backtrack(list, tempList, nums, used);
            used[i] = false; 
            tempList.remove(tempList.size() - 1);
        }
    }
}
```

### References
- https://zhuanlan.zhihu.com/p/30339183

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
