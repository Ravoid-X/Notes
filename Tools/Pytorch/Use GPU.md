## .cuda()
```
# 将网络模型在gpu上训练
model = Model()
if torch.cuda.is_available():
	model = model.cuda()

# 损失函数在gpu上训练
loss_fn = nn.CrossEntropyLoss()
if torch.cuda.is_available():	
	loss_fn = loss_fn.cuda()

# 数据在gpu上训练
for data in dataloader:                        
	imgs, targets = data
    if torch.cuda.is_available():
        imgs = imgs.cuda()
        targets = targets.cuda()
```