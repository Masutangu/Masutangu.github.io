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

```
class Solution {
public:

    void transform(int row, int idx, int& x, int& y) {
        int base = ceil(idx / (2 * row - 2.0));
        int mod = idx % (2 * row - 2);
        int offset = mod - row + 1 > 0 ?  mod - row + 1 : 0;
        y = base * (row - 1)  + offset;
        x = mod > row - 1 ? 2 * row - 2 - mod : mod;  
    }

    string convert(string s, int numRows) {
        if (numRows == 1) {
            return s;
        }
        int x, y, numCols = 0;
        transform(numRows, s.size() - 1 , x, numCols);
        
        numCols++;
        
        char** matrix = new char*[numRows];
        
        for (int i = 0; i < numRows; i++) {
            matrix[i] = new char[numCols];
            for (int j = 0; j < numCols; j++) {
                matrix[i][j] = '\0';
            }
        }
        
        char result[100000];
        
        for (int i = 0; i < s.size(); i++) {
            transform(numRows, i, x, y);
            matrix[x][y] = s[i];
        }
        int idx = 0;
        
        for (int i = 0; i < numRows; i++) {
            for (int j = 0; j < numCols; j++) {
                if (matrix[i][j] != '\0') {
                    result[idx++] = matrix[i][j];
                }
            }
        }
        return string(result);
        
    }
};
```

# Reverse Integer

Reverse digits of an integer.

Example1: x = 123, return 321
Example2: x = -123, return -321

要点在于如何检测 overflow：

```c
int reverse(int x) {
        int ans = 0;
        while (x) {
            int temp = ans * 10 + x % 10;
            if (temp / 10 != ans)
                return 0;
            ans = temp;
            x /= 10;
        }
        return ans;
    }
};
```


# String to Integer (atoi)

Implement atoi to convert a string to an integer.

关键点 去除空格，检测溢出：

```c++
class Solution {
public:
    int myAtoi(string str) {
        if (str.size() == 0) {
            return 0;
        }
        // 去首尾空格
        str.erase(0, str.find_first_not_of(' '));
        str.erase(str.find_last_not_of(' ') + 1);
        
        long num = 0;
        int sign = 1;
        int idx = 0;
        
        if (str[idx] == '+' || str[idx] == '-') {
            sign = (str[idx] == '-' ? -1 : 1);
            idx++;
        }
        
        for (; idx < str.size(); idx++) {
            if (str[idx] < '0' || str[idx] > '9') {
                break;
            }
            
            num = 10 * num + str[idx] - '0';
            
            // 检测溢出
            if (num * sign > INT_MAX) {
                return INT_MAX;
            } 
            if (num * sign < INT_MIN) {
                return INT_MIN;
            }
        }
        return num * sign;
        
    }
};
```
