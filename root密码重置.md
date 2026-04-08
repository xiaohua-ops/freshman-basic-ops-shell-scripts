# root密码重置

1. 重启 Linux
2. 按上键或者下键
3. 选择 Linux 内核

![img](https://ycn1pzibbi1s.feishu.cn/space/api/box/stream/download/asynccode/?code=YjVhY2RhNzQ1OGI4YWRmMTBkMDg0ZDI2OGNmMTA4NTJfdElrdUJhZmlWaFJkVUU0amp6eEY5R1Y1bXR4MTNUenhfVG9rZW46RWRPZGJyOW9Eb1hXMWd4bzdNcGM0bnJPbjlnXzE3NzU2NTAzODk6MTc3NTY1Mzk4OV9WNA)

1. 输入 e
2. 输入 rd.break

![img](https://ycn1pzibbi1s.feishu.cn/space/api/box/stream/download/asynccode/?code=NWM0YzNmMjVkNTAzMTI5ODY4NjBmNDYxZWJkZTUwNGNfTFpaSTVwc0VMQ0lpYVRjSVY5Z294VmxxdHhsa3dFRzRfVG9rZW46SzlMWGJ3MUVtb1lZU3J4dllJdWN2TVdUbnJnXzE3NzU2NTAzODk6MTc3NTY1Mzk4OV9WNA)

1. Ctrl + x 键

![img](https://ycn1pzibbi1s.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWVhMjAxOTA3ZmM5ZjI0NmRmY2VmYjc1YmY2MTk5ZTBfeE1hMlNmZTZDZkl4MG9ZODcydnk4WWt1aUxVbElwZUhfVG9rZW46SkR2Q2J1SGxZb093V1N4U3JLNWNlMFpnbnFYXzE3NzU2NTAzODk6MTc3NTY1Mzk4OV9WNA)

1. 然后按照以下命令进行 root 密码重置

```Shell
mount -o remount,rw /sysroot # 重新挂载根分区
chroot /sysroot # 切换根分区
passwd # 按回车键，然后输入两遍密码
或者
echo 密码|passwd --stdin root
touch /.autorelabel
exit
reboot
```

