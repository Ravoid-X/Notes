## 区间后缀极大位置计数
### 问题
<img src="../../../pic/C-Lang/Algorithm/Data Structure/mono_stack_exp1.png" style="width:600px;padding:10px;"/>

### 解题思路
1. 使用单调栈预计算下一个严格更大数组
2. 差分更新：对一个区间的更新操作转化为对差分数组两个位置的更新操作\
（1）假设想给区间 [L, R] 中的每个元素都加 1\
（2）a[L] 处加 1，会导致 a[L] 和 a[L-1] 的差值增加 1。所以让 d[L] += 1，同理 d[R+1] -= 1。\
（3）多个区间的更新操作，时间复杂度从 O(N×更新次数) 降为 O(更新次数)。
3. 计算前缀和：假设贡献区间是 [L, R]\
（1）执行 diff[L]++。表示从 L 开始，所有后续窗口的后缀最大值数量都增加了 1\
（2）执行 diff[R+1]--。表示从 R+1 开始，a[i] 的贡献结束了
### 程序
```
vector<int> next(n, n);
stack<int> s;
for (int i = n - 1; i >= 0; --i) {
    while (!s.empty() && a[s.top()] <= a[i]) {
        s.pop();
    }
    if (!s.empty()) {
        next[i] = s.top();
    }
    s.push(i);
}
vector<int> diff(n + 1, 0);
for (int i = 0; i < n; ++i) {
    int l = max(0, i - k + 1);
    int r = min(i, next[i] - k);
    if (l <= r) {
        diff[l]++;
        if (r + 1 <= n) {
            diff[r + 1]--;
        }
    }
}
int count = 0;
for (int i = 0; i <= n -k; ++i) {
    count += diff[i];
    cout << count << '\n';
}
```