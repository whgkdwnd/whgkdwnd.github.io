---
title: 완전범죄
date: 2025-12-19
thumbnail: /assets/img/thumbnail/paper_box.jpg
tags:
  - 레벨_2
  - dp
---
풀이 과정
	처음에는 그리디 알고리즘으로 풀려고 하였다.
	A가 훔치는 값은 내림차순 B가 훔치는 값은 내림차순으로 정렬하여 B가 가능한 한 모두 가져가고
	나머지 값에 대해서 A의 값을 판단하는 알고리즘을 짰지만,
	특정 케이스에서 최적해를 보장하지 못하는 경우가 나타나였다.
	``` ```
	이에 따라 DP로 풀이 방향을 선회 하였다.
	가방 문제를 응용하여 풀수있다고 판단,
	A가 훔칠때를 가치
	B가 훔칠때를 무게로 설정하여 문제를 풀었다.
	``` ```
	`dp[j]`를  
	“무게를 정확히 j만큼 사용했을 때 얻을 수 있는 최대 가치”로 정의하였고,  
	도달 불가능한 상태는 `-1`로 표시하였다.  
	또한, 이전 상태가 존재하는 경우(`dp[j-w] != -1`)에만 DP 갱신을 수행함으로써  
	존재하지 않는 상태에서 값이 생성되는 것을 방지하였다.
	``` ```
	최종적으로는 `j ≤ m` 범위 내에서 조건을 만족하는 DP 값 중 최대를 선택하여  
	문제의 요구 조건을 만족하는 해를 구하였다.
코드
```cpp
#include <string>
#include <vector>
#include <algorithm> 
using namespace std;



int solution(vector<vector<int>> info, int n, int m) {
	vector<int> weight(info.size());// 무게
	vector<int> value(info.size());// 가치
	vector<int> dp(info.size() + m, -1);
	dp[0] = 0;
	for (int i = 0; i < info.size(); i++)
	{
		weight[i] = info[i][1];
		value[i] = info[i][0];
	}
	for (int i = 0; i < info.size(); i++) {
		int v = value[i];
		int w = weight[i];

		for (int j = m; j >= weight[i]; j--) {
			if (dp[j - w] == -1) {
				continue;
			}
			dp[j] = max(dp[j], dp[j - w] + v);
		}
	}
	int max = -1;
	for (int i = 0; i < m; i++) {
		if (dp[i] > max) {
			max = dp[i];
		}
	}
	dp[m] = max;
	int sum = 0;
    int b_all_check =0;
	for (int i = 0; i < info.size(); i++) {
		sum += info[i][0];
        b_all_check += weight[i];
	}
    if(b_all_check < m){
        return 0;
    }
	sum -= dp[m];
	if (sum >= n) {
		return -1;
	}
	return sum;
}
```