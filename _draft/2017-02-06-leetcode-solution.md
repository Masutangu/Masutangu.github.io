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