### 问题
计算最长回文串
### 解题思路
1. 将原串插入分隔符，形如 $t=\#c1\#c2\#……\#cn\#$，分隔符为原字符串中不会出现的字符
2. 核心变量\
（1）center：当前已知回文最长右边界对应的中心；\
（2）right：该回文的最右端位置\
（3）$P[i]$：以位置  为中心的“回文半径”（包含分隔符）
3. 初始化 $P[i]=min(p[2*center-i], right-i)$，避免了从 1 开始暴力扩展
4. 中心扩展，$while(s[i-p[i]] == s[i+p[i]] && ... ) ++p[i]$
5. 若超出 right，则更新 center 和 right
### 程序
```
string s = "#";
for (char c : s_input) {
    s += c;
    s += '#';
}
int n=s.size();
vector<int> p(n, 1);
int center=0,right=0;
for (int i = 0; i < n; ++i) {
    if(i<right)  p[i]=min(p[2*center-i],right-i);
    while(s[i-p[i]]==s[i+p[i]]&&i-p[i]>=0&&i+p[i]<=n-1)  ++p[i];
    if(p[i]>1) --p[i];//cout<<"i="<<i<<" p[i]="<<p[i]<<endl;
    if (i + p[i] > right) {
        center = i;
        right = i + p[i];
    }

}
cout<<*max_element(p.begin(),p.end())<<endl;
```