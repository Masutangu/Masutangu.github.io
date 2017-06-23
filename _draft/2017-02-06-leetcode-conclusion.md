---
layout: post
date: 2017-02-06T08:55:19+08:00
title: Leetcode 
category: 算法
---


#

```c++

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


```c++
class Solution {
public:
    string longestPalindrome(string s) {
        char longest_str[1001];
        memset(longest_str, 0, sizeof(longest_str));
        int max_length = 0;
        
        int left_idx = 0;
        int right_idx = 0;
        int length = -1;
        int start_idx = -1;
        
        for (int i = 0; i < s.size(); i++) {
            left_idx = i;
            right_idx = i + 1;
            
            length = 1;
            start_idx = left_idx;
            
            while (left_idx >= 0 && right_idx < s.size() && s[left_idx] == s[right_idx]) {
                length = right_idx - left_idx + 1;
                start_idx = left_idx;
                left_idx--;
                right_idx++;
            }
            
            if (length > max_length) {
                max_length = length;
                strncpy(longest_str, s.c_str() + start_idx, max_length);
                longest_str[max_length] = '\0';
            }
        }
        
        for (int i = 1; i < s.size(); i++) {
            left_idx = i - 1;
            right_idx = i + 1;
            
            length = 0;
            start_idx = left_idx;
            
            while (left_idx >= 0 && right_idx < s.size() && s[left_idx] == s[right_idx]) {
                length = right_idx - left_idx + 1;
                start_idx = left_idx;
                left_idx--;
                right_idx++;
            }
            
            if (length > max_length) {
                max_length = length;
                strncpy(longest_str, s.c_str() + start_idx, max_length);
                longest_str[max_length] = '\0';
            }
        }
        
        return string(longest_str);
        
    }
};
```


```
class Solution {
public:
    int longestValidParentheses(string s) {
        stack<int> st;
        int max_len = 0;
        
        for (int i = 0; i < s.size(); i++) {
            if (s[i] == ')'  && !st.empty() && s[st.top()] == '(') {
                st.pop();
                int start = st.empty() ? -1 : st.top();
                max_len = max(i - start, max_len);
            } else {
                st.push(i);
            }
        }
        
        return max_len;
    }
};
```

```
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


```
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int start = 0, end = nums.size(), mid;
        
        while (start < end) {
            mid = (start + end) / 2;
            if (nums[mid] == target) {
                return mid;
            } else if (nums[mid] > target) {
                end = mid;
            } else {
                start = mid + 1;
            }
        }
        
        return start;
    }
};
```

```
/*
class Solution
{
public:
    int firstMissingPositive(int A[], int n)
    {
        for(int i = 0; i < n; ++ i)
            while(A[i] > 0 && A[i] <= n && A[A[i] - 1] != A[i])
                swap(A[i], A[A[i] - 1]);
        
        for(int i = 0; i < n; ++ i)
            if(A[i] != i + 1)
                return i + 1;
        
        return n + 1;
    }
};
*/
class Solution {
public:
    int firstMissingPositive(vector<int>& nums) {
        if (nums.empty()) {
            return 1;
        }
        int temp;
        for (int i = 0; i < nums.size(); i++) {
            if (nums[i] > nums.size() || nums[i] <= 0) {
                continue;
            }
            if (nums[i] != (i + 1) && nums[i] != nums[nums[i] - 1]) {
                temp = nums[nums[i] - 1];
                nums[nums[i] - 1] = nums[i];
                nums[i] = temp;
                i--;
            }
        }
        
        for (int i = 0; i < nums.size(); i++) {
            if (nums[i] != (i + 1)) {
                return i + 1;
            }
        }
        return nums.back() + 1;
    }
};
```

```
class Solution {
public:
    string multiply(string num1, string num2) {
        reverse(num1.begin(), num1.end());
        reverse(num2.begin(), num2.end());
        
        string result(num1.size() + num2.size() + 2, '0');
        
        int carry = 0;
        int res;
        for (int i = 0; i < num1.size(); i++) {
            for (int j = 0; j < num2.size(); j++) {
                res = (num1[i] - '0') * (num2[j] - '0') + (result[i + j] - '0') + carry;
                carry = res / 10;
                result[i + j] = (res % 10 + '0');
            }
            res = result[i + num2.size()] - '0' + carry;
            carry = res / 10;
            result[i + num2.size()] = (res % 10 + '0');
        }
        result[num1.size() + num2.size()] += carry;
        
        int i = result.size() - 1;
        while (i >= 1 && result[i] == '0') {
            i--;
        }
        
        string sub_str = result.substr(0, i + 1);
        reverse(sub_str.begin(), sub_str.end());
        
        return sub_str;
    }
};
```

```c++
class Solution {
public:
    bool isMatch(string s, string p) {
        int last_s_idx = -1, last_p_idx = -1, p_idx = 0;
        
        for (int i = 0; i < s.size(); i++) {
            if (p_idx < p.size() && s[i] == p[p_idx] || p[p_idx] == '?') {
                p_idx++;
            } else if (p_idx < p.size() && p[p_idx] == '*') {
                last_s_idx = i;
                last_p_idx = p_idx;
                i--;
                p_idx++;
            } else {
                if (last_s_idx != -1 && last_p_idx != -1) {
                    i = last_s_idx++;
                    p_idx = last_p_idx + 1;
                } else {
                    return false;
                }
            }
        }
        while (p_idx < p.size() && p[p_idx] == '*') {
            p_idx++;
        }
        if (p_idx >= p.size()) {
            return true;
        }
        
        return false;
    }
};
```