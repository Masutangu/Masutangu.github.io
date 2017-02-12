---
layout: post
date: 2017-02-06T08:55:19+08:00
title: Leetcode 
category: 算法
---

# Longest Substring Without Repeating Characters

Given a string, find the length of the longest substring without repeating characters.

Examples:

Given "abcabcbb", the answer is "abc", which the length is 3.

Given "bbbbb", the answer is "b", with the length of 1.

Given "pwwkew", the answer is "wke", with the length of 3. Note that the answer must be a substring, "pwke" is a subsequence and not a substring.

```c++
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        int max_length = 0;
        map<char, int> position;
        map<char, int>::iterator it;
        
        for(int i = 0; i != s.size(); i++) {
            it = position.find(s[i]);
            if (it == position.end()) {
                position.insert(pair<char,int>(s[i], i));
            } else {
                max_length = max_length > position.size() ? max_length : position.size();
                i = it->second;
                position.clear();
            }
        }
        if (!position.empty()) {
            max_length = max_length > position.size() ? max_length : position.size();
        }
        return max_length;
    }
};
```

由于是字符串，可以不使用map用一个256长度的数组：

```
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        int max_length = 0;
        int position[256];
        memset(position, 0 ,sizeof(position));
        int curr_length = 0;
        
        
        for(int i = 0; i != s.size(); i++) {
            int index = s[i];
            
            if (position[index] == 0) {
                position[index] = i + 1;
                curr_length++;
            } else {
                max_length = max_length > curr_length ? max_length : curr_length;
                i = position[index] - 1;
                memset(position, 0 ,sizeof(position));
                curr_length = 0;
            }
        }
        
        max_length = max_length > curr_length ? max_length : curr_length;
        
        return max_length;
    }
};
```

更简洁的：

```
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        vector<int> position(256, -1);
        int max_length = 0;
        
        int start = -1;
        
        for (int i = 0; i < s.size(); i++) {
            if (position[s[i]] > start) {
                start = position[s[i]];
            }  
            position[s[i]] = i;
            max_length = max_length > i - start ? max_length : i - start;
        }
        
        return max_length;
    }
};
```

# Median of Two Sorted Arrays

There are two sorted arrays nums1 and nums2 of size m and n respectively.

Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).

Example 1:
nums1 = [1, 3]
nums2 = [2]

The median is 2.0
Example 2:
nums1 = [1, 2]
nums2 = [3, 4]

The median is (2 + 3)/2 = 2.5

```c
double GetMedian(int* num1, int n, int* num2, int m, int idx) {
    if (n <= 0) {
        return num2[idx];
    } else if (m <= 0) {
        return num1[idx];
    } 
    
    int mid1 = (n - 1) / 2;
    int mid2 = (m - 1) / 2;
    
    if (mid1 + mid2 < idx) {
        if (num1[mid1] > num2[mid2]) {  
            // drop num2[:m/2]
            return GetMedian(num1, n, num2 + mid2 + 1, m - mid2 - 1, idx - mid2 - 1);
        } else {  
            // drop num1[:n/2]
            return GetMedian(num1 + mid1 + 1, n - mid1 -1, num2, m, idx - mid1 -1);
        }
    } else {
        if (num1[mid1] > num2[mid2]) { 
            // drop num1[n/2:]
            return GetMedian(num1, mid1, num2, m, idx);
        } else {
            // drop num2[m/2:]
            return GetMedian(num1, n, num2, mid2, idx);
        }
    }
}

double findMedianSortedArrays(int* nums1, int nums1Size, int* nums2, int nums2Size) {
    if ((nums1Size + nums2Size) % 2 == 0) {
        return (GetMedian(nums1, nums1Size, nums2, nums2Size, (nums1Size + nums2Size)/2) + 
               GetMedian(nums1, nums1Size, nums2, nums2Size, (nums1Size + nums2Size)/2 - 1))/2;
    } else {
        return GetMedian(nums1, nums1Size, nums2, nums2Size, (nums1Size + nums2Size)/2);
    }
    
}
```