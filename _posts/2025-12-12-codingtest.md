---
layout: post
title: "택배 상자 꺼내기"
date: 2025-12-13
categories: [notion, markdown]
---



# 코딩 테스트 기록

- 택배 상자 꺼내기
    
    [문제](https://school.programmers.co.kr/learn/courses/30/lessons/389478)
    
    - 문제
        
        1~ `n`의 번호가 있는 택배 상자가 창고에 있습니다. 당신은 택배 상자들을 다음과 같이 정리했습니다.
        
        왼쪽에서 오른쪽으로 가면서 1번 상자부터 번호 순서대로 택배 상자를 한 개씩 놓습니다. 가로로 택배 상자를 `w`개 놓았다면 이번에는 오른쪽에서 왼쪽으로 가면서 그 위층에 택배 상자를 한 개씩 놓습니다. 그 층에 상자를 `w`개 놓아 가장 왼쪽으로 돌아왔다면 또다시 왼쪽에서 오른쪽으로 가면서 그 위층에 상자를 놓습니다. 이러한 방식으로 `n`개의 택배 상자를 모두 놓을 때까지 한 층에 `w`개씩 상자를 쌓습니다.
        
    - 조건
        - 2 ≤ `n` ≤ 100
        - 1 ≤ `w` ≤ 10
        - 1 ≤ `num` ≤ `n`
    - 내 해답
        
        이중 배열  a[배열의 높이(n/w+1)][입력받은 w]로 구성한다
        
        배열의 높이 - num의 높이를 한 이후
        
        a[n/w][num]의 위치가 유효한 숫자인지 판별
        
        유효숫자일 경우 return 배열의 높이 - num의 높이  + 1
        
        유효숫자가 아닐경우 return 배열의 높이 - num의 높이
        
        - 코드
            
            int solution(int n, int w, int num) {
            int count = 1;
            int total_floor = (n / w) + 1;
            int** a = new int* [total_floor];// 배열 생성
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
            
            ```
            }
            int floor = (num) / w;
            if ((num) % w == 0) {// 내림을 기준으로 설계했는데 나누어 떨어지면 강제로 1을 낮춤
                floor--;
            }
            int location = 0;
            for (int i = 0; i < w; i++) {//원하는 수 위치 찾기
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
            
            ```
            
            }