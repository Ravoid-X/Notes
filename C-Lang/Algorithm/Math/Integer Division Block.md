计算 $S=\sum _{i=1} ^n(k\  mod\ i)$，运算时间为 $O(n)$，可运用整除分块算法来优化，达到$O(\sqrt n)$
<img src="../../../pic/C-Lang/Algorithm/Math/integer division block.png" style="width:400px;padding:10px;"/>
<img src="../../../pic/C-Lang/Algorithm/Math/integer division block2.png" style="width:400px;padding:10px;"/>
```
size_t r = min(n,k);
long long cnt=(n-r)*k;
long long i=1;
while(i<=r){
    size_t q = k / i;
    size_t R = min(r, k / q);
    size_t ct = R - i + 1;
    long long sum_i = (i + R) * ct / 2;
    cnt += ct * k - q * sum_i;
    i = R + 1;
}
```