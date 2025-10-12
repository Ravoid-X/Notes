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
int findMax(const vector<int>& rods) {
    int totalSum = accumulate(rods.begin(), rods.end(), 0);
    // dp[diff + offset] 存储当支架1长度 - 支架2长度 = diff 时，支架1的最大长度
    // 长度差的范围是 [-totalSum, totalSum],使用 offset 来将索引映射到非负数范围
    int offset = totalSum;
    vector<int> dp(2 * totalSum + 1, -1);
    dp[offset] = 0;

    for (int rod_len : rods) {
        // 创建一个副本，以避免在同一步迭代中混用新旧状态
        vector<int> current_dp = dp;
        for (int diff_idx = 0; diff_idx <= 2 * totalSum; ++diff_idx) {
            // 如果该状态是可达的
            if (current_dp[diff_idx] != -1) {
                int sum1 = current_dp[diff_idx] + rod_len;
                int new_diff_idx1 = diff_idx + rod_len;
                if (new_diff_idx1 <= 2 * totalSum) {
                    dp[new_diff_idx1] = max(dp[new_diff_idx1], sum1);
                }
                sum1 = current_dp[diff_idx];
                int new_diff_idx2 = diff_idx - rod_len;
                if (new_diff_idx2 >= 0) {
                    dp[new_diff_idx2] = max(dp[new_diff_idx2], sum1);
                }
                // 选择3 (不使用当前钢筋) 的情况被隐式处理了，
                // 因为 dp 数组的值会从上一步继承下来。
            }
        }
    }
    return (dp[offset] > 0) ? dp[offset] : 0;
}
```