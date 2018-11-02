# 
### 准备工作
找出程序的可执行文件路径

- root# `ps aux | grep Snapchat`

列出所有的 Frameworks

- xx.app root#  `ls Frameworks`

把所需要解密的库,复制出来

- `scp -P 2222 -r root@localhost:/var/....xx.app/Frameworks/xx`
- `scp -P root@10.0.0.3:/var/....xx.app/Frameworks/xx`

获取可执行文件的头部信息

- `otool -hf xxx(可执行文件)`

 ![图1.png](https://upload-images.jianshu.io/upload_images/1009061-ee4b5e75184e14e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


获取64位的加密信息(arm64)

-  `otool -arch arm64 -l 可执行文件 | grep crypt`

   ![图2.png](https://upload-images.jianshu.io/upload_images/1009061-83ba28c325a4dfc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### LLDB 附加

获取解密的目标文件加载模块的基地址

- `im list -o -f 可执行文件` 

 ![图3.png](https://upload-images.jianshu.io/upload_images/1009061-f2fc22243c120c32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 
获取解密后的二进制数据
 
- `memory read --force --outfile ~/Desktop/dumpoutput --binary --count cryptsize cryptoff+基地址`
    
     - cryptsize来自加密信息,cryptoff 也是来自加密信息
    
![图4.png](https://upload-images.jianshu.io/upload_images/1009061-f8cf996b43126051.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> ~/Desktop/dumpoutput 是保存到本地计算机的目标可执行文件,这个文件包含的是解密后的数据

### 修复文件

 因为 dump 出来的文件都没有 Mach-O 文件头,所以要先把  dump 出来的数据写会原来加密的文件,以替换原来加密的部分.
 
- `dd seek=20086784 bs=1 conv=notrunc if=/Users/wyp/Desktop/dumpoutput of=./可执行文件名`

![图5.png](https://upload-images.jianshu.io/upload_images/1009061-adab728f4ac40613.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 20086784是20070400+16384的计算结果.20070400是之前获取的 arm64架构的偏移值 offset,16384 是加密数据的 crypoff, 二者相加,得到了写入加密数据在文件中的偏移值

替换之后,使用 lipo 从 FAT 文件中提取 arm64 架构的文件,然后将 arm64_tmp 拖到 MachOView 中,修改 cryptid 的值为00000000,然后就完成了 framework 的解密.

- `lipo 可执行文件 -thin arm64 -output arm64_tmp`

![图6.png](https://upload-images.jianshu.io/upload_images/1009061-669c2aabcaf88b77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图7.png](https://upload-images.jianshu.io/upload_images/1009061-e74575110d644c28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

查看64位的加密信息

![图8.png](https://upload-images.jianshu.io/upload_images/1009061-9b6c42467d3af07d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


完成

> 部分 framework, 通过 image li 查不到基地址，解密的方式待续！