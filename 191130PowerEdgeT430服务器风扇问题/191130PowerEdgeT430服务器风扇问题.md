# PowerEdge T430 服务器风扇问题

## 背景

办公室新采购了一台戴尔的 PowerEdge T430 服务器，加装了四块 Titan RTX 显卡，给他们用来深度学习。虽然我不知道什么是深度学习，但是这个嗷嗷贵的服务器看起来就很高端，就很有深度，所以我相信他会帮助同学们搞好深度学习。  
![深度学习](./screenshot1.png)
作为~~技术总监~~网管，我就负责把机器拆箱，扛到机柜上，各种插好电线，OK，四路泰坦一次点亮，万德福，没毛病，老铁陆陆陆。  
不愧是贵的电脑，风扇呜呜地，差点没把 200 来斤的我给吹飞，真是给力。不过这系统初始化完成了，进来分配一下用户权限，不至于还这么嗷嗷吹吧，多费电啊。  
机智地我掏出 Pixel 下意识地谷歌了一下`T640 风扇`，第一页就有一篇 CSDN 的博客，说 Dell T640 安装显卡后风扇转速不降低的问题。

## 问题

我眉头一皱，发现问题没有那么简单，估计是硬件不是很兼容导致的，就从机房爬出来接着谷歌。经过一番搜索，看了戴尔论坛里的一些帖子、各种博客的文章，发现这真的是一个问题。针对这个问题，各种颜色的皮肤，各种颜色的头发，嘴里念的说的都是 CNM。  
事情是这个样子的，戴尔的服务器有一个很高端的功能，叫做`Multi-Vector Cooling`，说人话就是可以用多个指标来控制散热风扇。这个策略下有一个功能就是`PCIESlotLFM`，就是根据 PCI 插槽上的设备来控制散热风扇。也就是说，PCI 插槽上的设备用的猛了，就使劲吹抓紧散热，不咋用了，就轻点吹省省电。  
想法是美好的，可是实际上支持的显卡貌似不是很多，对于这种不支持的显卡，戴尔采取了一刀切的方式，就是满功率吹。可以，这很懒政。正是因为这个原因，才让我们的服务器风扇一直可劲儿吹。

## 解决

如果就这样了，那可不行啊，我们老板贼鸡儿抠，要是知道了这机器的电费比我的工资还高，不得把我给 Fire 喽。我得想想办法。  
就在第一次搜到的那个文章中，博主提到了用一个叫`radacm`的工具可以进行设置。这又是什么鬼，看好多论坛里也提到了，又一顿搜，哦，原来是戴尔的管理工具。在戴尔官方的支持中，可以下载到，下载项目是`Dell EMC iDRAC Tools for Linux`，tar 解压缩一下，就有了一个`iDRACTools`目录，在里面的`radacm`子目录中，有`install_racadm.sh`。看到这个就放心了，看什么文档，就是 sudo，大不了机器坏了，反正在保怕什么。果然，安装过后可执行文件就拷贝到`/opt/dell/srvadmin/sbin/racadm`，这个工具的具体用法，可以参考[戴尔的官方文档](https://www.dell.com/support/manuals/cn/zh/cnbsd1/idrac9-lifecycle-controller-v3.30.30.30/idrac_3.30.30.30_racadm/introduction)，对于`System. pcieSlotLFM`，可以进行各种属性的 set 或 get。我先 get 了一下，发现四个显卡被插在了 1、3、6、8 四个插槽中，他们的 LFMMode 都是 Automatic，也就是 0 级。按照文档的说明，我设置为了 2 级（这里文档好像是有笔误，写成了 0 级、1 级和 1 级，我提了反馈意见，不知道会不会领到奖金……），果然就冷静下来了。

## 后记

我又深入了解了一下，这里控制的只是机箱内部的风扇，各个显卡的风扇还是由硬件自己来控制，CPU 的风扇也不会因此受到影响。  
为了能够在同学们确实进行深度计算时候能够加强散热，我写了一个 Python 脚本，根据`nvidia-smi`的查询结果动态地在两种模式之间切换，实现了电力消耗和温度控制的平衡。~~这才是真正的人工智能~~什么是吹逼？这才是吹逼。

```python
import os
import time

def get_temp():
    cmd = " nvidia-smi -q -d Temperature | grep 'GPU Current' | awk '{print $5}' "
    result = os.popen(cmd)
    result = result.read()
    tmps = result.strip().split('\n')

    tmp = 0
    for s in tmps:
        t = float(s)
        if t > tmp:
            tmp = t

    return tmp

def set_fan(low = True):
    n = 0
    if low == True:
        n = 2

    for i in (1, 3, 6, 8):
        cmd = "/opt/dell/srvadmin/sbin/racadm set System.PCIESlotLFM.%d.LFMMode %d" % (i, n)
        print("执行命令: " + cmd)
        os.popen(cmd)

last_low = False

while True:
    cur_low = True
    temp = get_temp()
    s = "冷静"
    if temp > 83.0:
        cur_low = False
        s = "疯狂"

    if cur_low != last_low:
        last_low = cur_low
        set_fan(cur_low)
        t = time.asctime(time.localtime(time.time()))

        print(t + "切换为" + s)

    time.sleep(60)
```

## 再后记

本来以为这种两档切换的不够高端，结果运行几天发现，在计算的时候，即使是风扇开满也会压不住，过一会儿机房就热起来了，服务器的高温报警就亮了。四路泰坦竟恐怖如斯！  
下一步的工作就是劝老板大冬天的也要给机房开冷气了，太难了，还是写代码简单。。。
