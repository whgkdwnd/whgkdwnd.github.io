---
title: 유연 근무제
tags:
  - 반복문
  - 레벨_1
date: 2025-12-04
thumbnail: /assets/img/thumbnail/worklb.jpg
---



# 유연근무제

- [문제](https://school.programmers.co.kr/learn/courses/30/lessons/388351)

- 내해답
    
    개요:
    
    직원 한명을 붙잡고 직원의 출근 최대 시간 보다 늦게 출근했을경우 check++를 하여 check가 0일경우 선물++ 아닐경우 그냥 넘어가기.
    

코드
```cpp
#include <vector>

using namespace std;

int solution(vector<int> schedules, vector<vector<int>> timelogs, int startday) {
    int present = 0;
    for (int i = 0; i < schedules.size(); i++) {
        int set_max_time = schedules[i]+10;
        if (set_max_time % 100 >= 60) {
            set_max_time = set_max_time - 60 + 100;
        }
        int now_day = startday;
        int check = 0;
        for (int j = 0; j < 7; j++) {
            if (now_day == 6 || now_day == 7) {// 주말이면 넘겨라
                now_day++;
                if(now_day == 8){
                    now_day =1;
                }
                continue;
            }
            if (timelogs[i][j] <= set_max_time) {
                now_day++;
                continue;
            }
            else {
                check++;
                break;
            }
        }
        if (check == 0) {
            present++;
        }
    }
    return present;// 상품을 받을 지원의 수
```
