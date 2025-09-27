### 修改 rc-local
1. 进入目录\
`sudo gedit /lib/systemd/system/rc-local.service`
2. 添加 Install\
```
[Install]
WantedBy=multi-user.target
Alias=rc-local.service
```
### 创建 rc.local 文件
`touch /etc/rc.local`
### 修改 rc.local 权限
`sudo chmod +x /etc/rc.local`
### 启动服务
`sudo systemctl enable rc-local.service` 或\
`sudo systemctl start rc-local.service`
### 查看状态
`sudo systemctl status rc-local.service`