## 单个文件夹提交 message
```Bash
feat(folder name): message
```
## 防止 make 生成的文件被错误识别成代码
1. 在项目的根目录新建 `.gitattributes` 文件
2. 添加
```
*.txt linguist-language=C++
*.cmake linguist-language=C++
*.make linguist-language=C++
Makefile linguist-language=C++
```