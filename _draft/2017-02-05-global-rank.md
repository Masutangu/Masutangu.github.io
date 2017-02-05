---
layout: post
date: 2017-01-25T20:49:16+08:00
title: 全区排行榜
category: 算法
---

关卡排行榜：有百分之多少的人已经到达这个关卡？

```c
const int MAX_STAGE = 1000;

int stage_rank_list[MAX_STAGE + 1];

stuct stage_rank_info {
    int uid;
    int stage;     // or int score     
    int64 timestamp;  // 或其他排序条件
};

int lowest_realtime_score = 0;  // 高于这个分数就需要实时排序

map<int, stage_rank_info> stage_rank_map;  // key 为 uid

void load_stage() {
    for user in user_list {
        stage_rank_list[user->rank_stage_info.stage]]++;
    }
}
  
void update_stage(max_stage) {
   stage_rank_list[max_stage]++;
}

float get_percent(query_stage, pass_people, total_people) { 
    pass_people = 0;  // 引用
    total_people = 0;  // 引用

    for (stage = MAX_STAGE; stage >= 0; stage--) {
        if (stage > query_stage) {
            pass_people++;
        }
        total_people++;
    }
    return float(pass_people)/total_people;
}

float get_rank(stage_info, int* rank_list, vector<stage_rank_info> stage_rank_list) {
    int pass_people = 0;
    int total_people = 0;

    if (pass_people < 10000) {
        stage_rank_map[uid] = stage_info;

        for stage_rank_info in stage_rank_map {
            if (stage_rank_info.stage < query_stage) {
                stage_rank_map.delete(stage_rank_info);
            } else {
                stage_rank_list.push_back(stage_rank_info);
            }
        }
        sort(stage_rank_info);
        lowest_realtime_score ＝ stage_rank_info[stage_rank_info.size()-1];  // 显示实时排名的最低分数
    }
}

// 这部分分摊在各个gamesvr？以分数为key的不能缓存
struct stage_cache_info {
    int stage;
    int64 expire_time;
    float percent;
    vector<stage_rank_info> stage_rank_list;
}

map<int, stage_cache_info> cache_info;  // key 为 uid，value为缓存信息

float get_rank_with_cache(stage_info, int* rank_list, vector<stage_rank_info> stage_rank_list) {
    // 如果分数超过之前实时显示的最低分数，则需要重新排一下了
    if stage_info.uid in cache_info && stage_info.score < lowest_realtime_score {
        if stage_info.stage < cache_info[stage_info.uid].stage ||
            now < cache_info[stage_info.uid].expire_time {
                stage_rank_list = cache_info[stage_info.uid].stage_rank_list;
                return cache_info[stage_info.uid].percent;
        }
    }

    float percent = get_rank(stage_info, rank_list, stage_rank_list);

    if (percent > 0.1) {  // 排名很靠后，可以缓存，不需要实时查询
        // 缓存起来
    }
    return percent;
}

```

