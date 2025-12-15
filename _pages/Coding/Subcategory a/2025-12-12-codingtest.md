---
title: 택배상자 꺼내기
tags:
  - 반복문
  - 레벨_1
date: 2025-12-04
thumbnail: /assets/img/thumbnail/paper_box.jpg
---


- 택배 상자 꺼내기
    
    [문제](https://school.programmers.co.kr/learn/courses/30/lessons/389478)
    - 내 해답
        
        이중 배열  a[배열의 높이(n/w+1)][입력받은 w]로 구성한다
        
        배열의 높이 - num의 높이를 한 이후
        
        a[n/w][num]의 위치가 유효한 숫자인지 판별
        
        유효숫자일 경우 return 배열의 높이 - num의 높이  + 1
        
        유효숫자가 아닐경우 return 배열의 높이 - num의 높이
        
코드
```cpp
#include <string>
#include <vector>

using namespace std;

int solution(int n, int w, int num) {
    int count = 1;
    int total_floor = (n / w) + 1;
    int** a = new int* [total_floor];
    for (int i = 0; i < total_floor; i++) {
        a[i] = new int[w];
        if (i % 2 == 0) {
            for (int j = 0; j < w; j++) {
                a[i][j] = count;
                count++;
            }
        }
        else
        {
            count += w-1;
            for (int j = 0; j < w; j++) {
                a[i][j] = count;
                count--;
            }
            count += w+1;
        }
        
    }
    int floor = (num) / w;
    if ((num) % w == 0) {
        floor--;
    }
    int location = 0;
    for (int i = 0; i < w; i++) {
        if (a[floor][i] == num) {
            location = i;
            break;
        }
    }
    int now_floor = floor + 1;
    if (a[total_floor - 1][location] > n) {// 맨 윗값이 오버한 값이냐를 물어보기
        return total_floor - now_floor;// oo 오버한 값이다
    }
    else
    {
        return total_floor - now_floor + 1;// 우리는 맨 위에 하나 더 있어
    }
}
```
