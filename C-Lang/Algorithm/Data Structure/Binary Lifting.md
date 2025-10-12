## 倍增
一种典型的用空间换时间的技巧，将任意大的信息拆分成若干个 2 的幂次项之和，从而实现快速查询。将一个 O(N) 的线性操作，通过预处理优化为 O(log N) 的操作
## 环形字符串跃迁
### 问题
<img src="../../../pic/C-Lang/Algorithm/Data Structure/bl_exp1.png" style="width:600px;padding:10px;"/>

### 解题思路
1. 先预处理得到每个位置最近的 0 位置（包括自身）
2. $next[i][j]$ 表示位置 i 走 $2 ^j$ 步后的落点
3. 将 k 进行二进制分解，即可得到最终落点
### 程序
```
vector<int> p(n);
vector<vector<int>> next(n, vector<int>(log_k));
if(count(s.begin(),s.end(),'0')==0){
    while (q--) {
        int t;
        ll k;
        cin >> t >> k;
        cout<<(t-1+k)%n+1<<endl;
    }
    return 0;
}
int cur_0;
for (int i = n - 1; i >= 0; --i) {
    if (s[i] == '0') {
        cur_0 = i;
        break;
    }
}
for (int i = 0; i < n; ++i) {
    if (s[i] == '0') {
        cur_0 = i;
    }
    p[i] = cur_0;
}
for (int i = 0; i < n; ++i) {
    int dist = (p[(i + m) % n]-i+n)%n;
    if (dist > 0 && dist <= m) {
        next[i][0] = p[(i + m) % n];
    } else {
        next[i][0] = (i + 1) % n;
    }
}
for (int j = 1; j < log_k; ++j) {
    for (int i = 0; i < n; ++i) {
        next[i][j] = next[next[i][j - 1]][j - 1];
    }
} 
while (q--) {
    int t;
    ll k;
    cin >> t >> k;
    --t;
    vector<bool> binary;
    while(k>0){
        binary.push_back((k&1)?true:false);
        k >>= 1;
    }
    for(short i=binary.size()-1;i>=0;i--){
        if(binary[i]){
            t=next[t][i];
        }
    }
    cout<<t+1<<endl;
}
```