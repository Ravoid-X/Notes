## 取整 `#include <cmath>`
1. `double floor(double x)`
返回小于或等于 x 的最大整数
2. `double ceil(double x)`
返回大于 x 的最小整数
## 求余
1. '%'后的符号与被除数相同，如要将结果映射到 [0,P]，使用 (a % P + P) % P，若 a<=P，可简化为 (a + P) % P