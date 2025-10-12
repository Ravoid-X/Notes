## 钢筋拼接
### 问题
假如有一个正整数数组，储存着一堆钢筋的长度，现要组成两个等长的支架，求最长的支架长度是多少
### 思路
1. 对于每一根钢筋，有三种选择：放到第一个支架上；放到第二个支架上；不使用。
2. 定义一维数组 dp，其中 dp[k] 表示当两个支架的长度差为 k 时，第一个支架可能的最大长度。
3. 由于长度差可正可负，用 offset = totalSum，将 dp 数组的索引映射到 [0, 2 * totalSum]
4. 遍历完所有钢筋后，dp[0 + offset] 的值就代表了当两个支架长度差为 0 时，第一个支架的最大长度。
### 程序
```
// 用于在栈中存储状态的结构体
struct State {                                                                   
    int index; 
    int sum1;
    int sum2;
};
int findMax(vector<int>& rods) {
    if (rods.empty()) {
        return 0;
    }
    // 优化：从大到小排序
    sort(rods.begin(), rods.end(), greater<int>());
    int totalSum = accumulate(rods.begin(), rods.end(), 0);
    int maxLength = 0;
    stack<State> s;
    // 压入初始状态，相当于第一次调用 dfs(0, 0, 0)
    s.push({0, 0, 0});
    while (!s.empty()) {
        // 取出当前状态进行处理
        State currentState = s.top();
        s.pop();
        int index = currentState.index;
        int sum1 = currentState.sum1;
        int sum2 = currentState.sum2;
        // 如果所有钢筋都已处理完毕
        if (index == rods.size()) {
            if (sum1 == sum2) {
                maxLength = max(maxLength, sum1);
            }
            // 到达叶子节点，继续处理栈中下一个状态
            continue; 
        }
        // --- 剪枝 ---
        if (sum1 > totalSum / 2 || sum2 > totalSum / 2) {
            continue;
        }
        // 将新状态压入栈中
        int currentRod = rods[index];
        // 为了模拟DFS的探索顺序，压栈的顺序与递归调用的顺序相反。
        // 递归版本会先深入探索“决策1”，所以我们最后把它压栈，以保证它被最先处理。
        s.push({index + 1, sum1, sum2});
        s.push({index + 1, sum1, sum2 + currentRod});
        s.push({index + 1, sum1 + currentRod, sum2});
    }
    return maxLength;
}
```