# 一、解析工具
## 1.1 readelf工具
windows下需要安装Cygwin

## 1.2 命令
1. readelf -h xx.so     查看so文件的头部信息
2. readelf -S xx.so     查看so文件的节(Section)头的信息
3. readelf  -l xx.so     查看so文件的段(Program)头的信息
4. readelf -a xx.so     查看so文件的所有信息