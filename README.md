# reproducibleopeneuler
可重复构建sig组通过一系列开源工具，使openEuler构建的软件能达到可重复构建的目标。

为此，我们在执行两次Open Build Service(OBS)重复构建时，主要打桩了以下两个参数，使用的开源软件是libfaketime ：  

* datetime
* hostname

通过unpack解压工具将不一致的包解压出结果，解压出的不一致结果通过(diffoscope)显式展示出具体不同点，用以后续消费分析，最终整改达到消除差异的目的。

可重复构建详细步骤：  
1. 安装 libfaketime：  
libfaketime源码地址： https://github.com/opensourceways/reproducible-builds-libfaketime  
```
make
make install
```
2. 设置 libfaketime 环境变量，打桩datetime和hostname  
```
echo 'export LD_PRELOAD=/usr/local/lib/faketime/libfaketimeMT.so.1' >> /etc/profile
echo 'export FAKETIME="2022-05-01 11:12:13"' >> /etc/profile
echo 'export FAKEHOSTNAME=fakename' >> /etc/profile
```
3. 额外设置在Python包中因 time 和 random numbers 导致的不可重复  
```
echo 'export SOURCE_DATE_EPOCH=1' >> /etc/profile  
echo 'export PYTHONHASHSEED=0' >> /etc/profile  
```
4. Sig组通过在libfaketime中设置对系统算法黑/白名单来达到消除系统差异的目的。 在黑名单模式中，libfaketime 只会劫持黑名单中的命令，而在白名单模式中，libfaketime 劫持除白名单中命令以外的所有命令。
   用户可以根据自己的需求，下载对应分支使用白名单或者黑名单模式：
```
# blacklist mode
export BLACKLIST=ls,make

# whitelist mode
export WHITELIST=ls,make
```
5. 在OBS构建系统中，hostname会导致rpm包的两次构建出现不一致. 因此,我们也在libfaketime中集成了挂接主机名的功能,目前该功能集成在白名单分支中。
```
export FAKENAME=fakename
```
6. 完成上诉配置步骤后，在OBS中构建两次软件包
7. 使用如下命令，使用unpacker将不一致的包进行解压  
```
python unpacker.py ${first package path} ${second package path}
```
8. 安装 diffoscope 并显示具体的差异点： 
```
yum install diffoscope  
diffoscope ${first file path} ${second file path} --html diff.html
```

