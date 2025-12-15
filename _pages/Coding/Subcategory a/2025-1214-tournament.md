---
title: 예상 대진표
tags:
  - 반복문
  - 레벨_1
date: 2025-12-14
thumbnail: /assets/img/thumbnail/tournament.jpg
---
[문제](https://school.programmers.co.kr/learn/courses/30/lessons/12985)

문제 풀이 방법
	자신이 짝수일 경우 자신의 번호/2를 부여 받고
	자신이 홀수일 경우 자신의 번호+1/2를 부여 받는다.
	그러므로 두 번호가 연속되어있는지 확인하고 아닐 경우 cnt를 1 올리고 새로운 번호를 부여 받는다
코드
```cpp
int solution(int n, int a, int b) {
    int cnt = 1;
    for (int i = 0; i < n / 2; i++) {
        if ((a + 1 == b && a % 2 == 1) || (b + 1 == a && b % 2 == 1)) {
            return cnt;
        }
        if (a % 2 == 1) a++;
        a /= 2;
        if (b % 2 == 1) b++;
        b /= 2;
        cnt++;
    }
}

```

