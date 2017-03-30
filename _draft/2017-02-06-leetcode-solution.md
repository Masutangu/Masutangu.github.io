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


# Regular Expression Matching

https://swtch.com/~rsc/regexp/regexp1.html

http://articles.leetcode.com/regular-expression-matching


```
bool isMatch(char* s, char* p) {
    if (*p == '\0') {
        if (*s == '\0') {
            return true;
        } else {
            return false;
        }
    }
    
    while (*s != '\0') {
        while (*p == '*') {
            p++;
        }
        if (*(p + 1) == '*') {
            while ((*p == '.' || *s == *p) && *s != '\0') {
                if (isMatch(s, p + 2)) {
                    return true;
                }
                s++;
            }
           return isMatch(s, p + 2); 
        } else {
            if (*p == '.' || *p == *s) {
                s++;
                p++;
            } else {
                return false;
            }
        }
    }
    
    while (*p == '*') {
        p++;
    }
    if (*p != '\0') {
        if (*(p + 1) == '*') {
            return isMatch(s, p + 2);
        }
        return false;
    } else {
        return true;
    }
}
```

```
bool isMatch(char* s, char* p) {
    while (*p == '*') {
        p++;
    }
    
    if (*p == '\0')
        return *s == '\0';
        
    if (*(p + 1) == '*') {
        while (*s != '\0' && (*p == '.' || *s == *p)) {
            if (isMatch(s, p + 2)) {
                return true;
            }
            s++;
        }
        return isMatch(s, p + 2);
    } else {
        return (*p == *s || (*s != '\0' && *p == '.')) && isMatch(s + 1, p + 1);
    }
    
}
```

# Container With Most Water

key: two pointer

```
int maxArea(int* height, int heightSize) {
    int water = -1;
    int idx1 = -1;
    int idx2 = -1;
    int pre_height = -1;
    
    for (int i = 0; i < heightSize; i++) {
        if (*(height + i) <= pre_height) {
            continue;
        } 
        pre_height = *(height + i);
        for (int j = i + 1; j < heightSize; j++) {
            if ((j - i) * (*(height + i)  < *(height + j) ? *(height + i) : *(height + j)) > water) {
                water = (j - i) * (*(height + i)  < *(height + j) ? *(height + i) : *(height + j));
                idx1 = i;
                idx2 = j;
            }
        }
    }
    return water;
}
```

```c
int maxArea(int* height, int heightSize) {
    int max_area = -1;
    int* start = height;
    int* end = height + heightSize - 1;
    
    while (start < end) {
        if ((end - start) * (*start > *end ? *end : *start) > max_area) {
            max_area = (end - start) * (*start > *end ? *end : *start);
        }
        if (*start > *end) {
            end--;
        } else {
            start++;
        }
    }
    return max_area;
}
```

# Integer to Roman

```c
class Solution {
public:
    string intToRoman(int num) {
        int nums[] = {1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1};
        string symbol[]={"M","CM","D","CD","C","XC","L","XL","X","IX","V","IV","I"}; 
        string roman;
        
        int i = 0;
        
        while (num > 0) {
            while (num >= nums[i]) {
                roman.append(symbol[i]);
                num -= nums[i];
            }
            i++;
        }
        
        return roman;
    }
    
};
```

# 3Sum

key: two pointer

# Letter Combinations of a Phone Number

key: backtracing

# Remove Nth Node From End of List

key: two pointer

# Merge k Sorted Lists

key: Heap

# Remove Element & Remove Duplicates from Sorted Array

key: two pointer

# Swap Nodes in Pairs

```c
ListNode* swapPairs(ListNode* head) {
    ListNode **pp = &head, *a, *b;
    while ((a = *pp) && (b = a->next)) {
        a->next = b->next;
        b->next = a;
        *pp = b;
        pp = &(a->next);
    }
    return head;
}
```

# Divide Two Integers

key: bit operation

```c++
class Solution {
public:
    int divide(int dividend, int divisor) {
        if (!divisor || (dividend == INT_MIN && divisor == -1))
            return INT_MAX;
        int sign = ((dividend < 0) ^ (divisor < 0)) ? -1 : 1;
        long long dvd = labs(dividend);
        long long dvs = labs(divisor);
        int res = 0;
        while (dvd >= dvs) { 
            long long temp = dvs, multiple = 1;
            while (dvd >= (temp << 1)) {  // 细节：dvd 和 temp 必须是 long，int 的话 temp << 1 可能溢出，造成死循环
                temp <<= 1;
                multiple <<= 1;
            }
            dvd -= temp;
            res += multiple;
        }
        return sign == 1 ? res : -res; 
    }
};
```

# Substring with Concatenation of All Words

key node: hash table

# Next Permutation

数学题

# Longest Valid Parentheses

key node: stack O(n)

```c++
class Solution {
public:
    int longestValidParentheses(string s) {
        int max_length = 0;
        stack<int> st;
        st.push(-1);
        for (int i = 0; i < s.size(); i++) {
            int top = st.top();
            if (top != -1 && s[top] == '(' && s[i] == ')') {
                st.pop();
                max_length = max(max_length, i - st.top());
            } else {
                st.push(i);
            }
        }
        return max_length;
    }
};
```

# Search in Rotated Sorted Array
keyword: 异或

```c++
class Solution {
public:
    int search(vector<int>& nums, int target) {
       int lo = 0, hi = int(nums.size()) - 1;
       while (lo < hi) {
            int mid = (lo + hi) / 2;
            // 456789123  67812345 3456789
            // 01->1 有且只有一个条件满足，或者全满足
            // nums[lo] > target && nums[start] > nums[mid] && target > nums[mid] -> lo = mid + 1
            // nums[lo] > target && nums[start] < nums[mid] && target < nums[mid] -> lo = mid + 1
            // nums[lo] < target && nums[start]  > nums[mid] && target < nums[mid] -> lo = mid + 1
            // nums[lo] < target && nums[start]  < nums[mid] && target > nums[mid] -> lo = mid + 1
            if ((nums[lo] > target) ^ (nums[lo] > nums[mid]) ^ (target > nums[mid]))
                lo = mid + 1;
            else
                hi = mid;
        }
        return lo == hi && nums[lo] == target ? lo : -1;
    }
    
};
```

# Search for a Range
keyword: binary search

```c++
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int start = 0, end = nums.size(), mid;
        
        while (start < end) {
            mid = (start + end) / 2;
            if (nums[mid] >= target) {
                end = mid;
            } else {
                start = mid + 1;
            }
        }
        int lo = start;
        
        start = 0;
        end = nums.size();
        
        while (start < end) {
            mid = (start + end) / 2;
            if (nums[mid] > target) {
                end = mid;
            } else {
                start = mid + 1;
            }
        }
        int hi = start;
        
        return lo == hi ? vector<int>{-1, -1} : vector<int>{lo, hi - 1};
    }
};
```


# Valid Sudoku   
keynode：别想复杂了

# Sudoku Solver
keynode：回溯

# Combination Sum
keynode: 回溯 别想复杂

# First Missing Positive
keynode: 索引

# Trapping Rain Water
keynode: two pointer / stack

```c++
class Solution {
public:
    int trap(vector<int>& height) {
        int start = 0, end = height.size() - 1, low = 0, level = 0, size = 0;
        
        while (start < end) {
            low = (height[start] < height[end]) ? height[start++] : height[end--];
            level = max(level, low);
            size += level - low;
        }
        return size;
    }
};
```