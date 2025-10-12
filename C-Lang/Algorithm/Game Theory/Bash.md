## 扩展巴什博弈
### 问题
<img src="../../../pic/C-Lang/Algorithm/Game Theory/bash_exp1.png" style="width:600px;padding:10px;"/>

### 解题思路
先手必败当且仅当 $N mod (L+R) <L$；否则先手必胜。
### 程序
```
vector<bool> res(T);
for(size_t i=0;i<T;i++){
    cin>>n>>l>>r;
    if((n%(l+r))>=l) res[i]=true;
    else res[i]=false;
}
for(size_t i=0;i<T;i++){
    if(res[i]) cout<<"YES"<<endl;
    else cout<<"NO"<<endl;
}
```