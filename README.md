# Unraid 使用ZFS文件系统 教程

如何无缝从TrueNAS CORE/SCALE 切换到Unraid

原文是在Notion上写的，查看Notion可以有更好的观看体验
https://miniature-ballcap-32c.notion.site/Unraid-ZFS-cfa3ca8dbab7426687eff601172a0535

# 前言（屁话）

从一开始接触NAS，使用的就是TrueNAS CORE以及其ZFS的文件系统。

ZFS在数据安全性上有许多优势，很多照片视频对我都有很多纪念意义，我对数据安全性非常看重。所以，数据安全是最高优先级，其次是性能。

在使用一段时间NAS后，接触到了Docker容器的概念，发现NAS不光光是存储，也是家庭媒体服务，代码服务的中心。所以在TrueNAS CORE中没有原生docker支持，只能使用CentOS虚拟机里使用docker应用非常不方便，且性能损失十分大。

于是，我发现了TrueNAS SCALE，有更高效的虚拟机以及原生支持轻量化的K8S（K3S）用于运行容器。在RC1版本更新时，我将系统切换到了SCALE，确实原生的容器应用更加高效且便利，K8S也带来Docker没有的更多特性。

但是SCALE作为ix在Linux上的第一次尝试，从RC1到第三个正式版，bug各种多，最重要的还是ZFS池的性能（或者是网络性能并不如TrueNAS CORE），随着应用增多，4C 32GB的配置已经闲的非常吃力。听到朋友说Unraid有ZFS插件，可以直接在主机上运行ZFS文件系统，于是打算尝试，发现中文教程也不多，就打算写一篇，帮助SCALE用户无缝跑路到Unraid。

> Unraid使用ZFS储存池的原理就是使用ZFS插件，安装ZFS的OpenZFS的开源项目，在Unraid使用命令行，创建导入并控制池，对于大部分人来说，池选项并不需要大量调整，所以只要几个简单的命令，就可以轻松使用。
> 

# 正片

## 添加应用源 && 设置语言

![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled.png)

1. 在图中的栏中输入下方的应用源后，点击 INSTALL，即可安装应用源。
    
    官方源（需要正常访问Github）：
    
    `https://raw.githubusercontent.com/Steini1984/unRAID6-ZFS/master/unRAID6-ZFS.plg`
    
    gitee国内镜像源：
    
    `https://gitee.com/BlueBuger/community.applications/raw/master/plugins/community.applications.plg`
    
2. 成功安装后，上方会多出一个栏APPS
3. 然后在 APPS →Language→简体中文语言包→点击Install （因为我这里已经安装，会显示为action）
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%201.png)
    
4. 安装完语言包后，我们还需要设置为中文。
5. 安装以下顺序就可以将语言设置为中文。
    
    SETTINGS → Display Settings → Language → 简体中文 → APPLY
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%202.png)
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%203.png)
    

PS：设置完可以在这里快速切换中英文

![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%204.png)

## 创建池

这里是给原来没有使用过ZFS，想要直接在Unraid上创建ZFS阵列的用户。

已经有ZFS阵列可以跳过此步骤

---

创建池需要对ZFS文件系统进行简单的了解。

这里推荐查看Sagit大佬的文章：[https://www.truenasscale.com/2021/12/20/353.html](https://www.truenasscale.com/2021/12/20/353.html)

下面的内容引用Sagit文章内容

阵列大致分为下面几种：

> 条带：与RAID0类似，不过可以不同大小的硬盘组（假设2块盘容量不一样，写入数据大小超过较小硬盘容量的两倍以后就没有只有大的硬盘的读写速度了）
> 

> 镜像（Mirror）：与RAID1类似，磁盘镜像，至少需要两个磁盘。
> 

> RAIDZ1：与RAID5类似，一重奇偶校验，至少需要三块磁盘；可以坏一块硬盘不丢数据。容量和速度为N-1(N为硬盘数量)
> 

> RAIDZ2：与RAID6类似，双重奇偶校验，至少需要四个磁盘；可以坏2块硬盘不丢数据。容量和速度为N-2(N为硬盘数量)
> 

> RAIDZ3：ZFS特有的，三重奇偶校验，至少需要5个磁盘；可以坏3块盘不丢数据。容量和速度为N-3(N为硬盘数量)
> 

1. 首先你需要知道自己硬盘在系统的标签名，Unraid中，可用在 主界面→ 磁盘 → 下拉栏中，找到你硬盘对于的标签名（入土橙色框）（我这里只使用一个盘作为示例，实际你需要记录自己所需要阵列对应的所有盘。）
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%205.png)
    
2. 在 应用 中搜索 ZFS 并安装如图对应的ZFS插件
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%206.png)
    
3. 打开Unraid的命令行
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%207.png)
    
4. 接下来我们就需要创建ZFS的池了
在命令行中输入：

zpool create -m /mnt/[池名字] [池名字] [阵列的类型] [磁盘的编号]

我们来解释一下这个命令的组成
zpool create ：是创建池
-m /mnt/[池名字] ：中的-m是指 将 储存池 挂载到后面系统中 /mnt/[池名字] 位置
[池名字] ：是指你想要创建的储存池的名字，可以是你自己的喜欢的名字
[阵列的类型] ： 在上面的介绍文章中有：mirror（镜像） raidz（RAIDZ）raidz2（RAIDZ2） raidz3（RAIDZ3）（如果什么都不输入即为条带）
[磁盘的编号]：即步骤1中记录的硬盘编号

示例：
1.需要创建一个 3盘 硬盘编号sdq sdx sdv ，条带（类似raid0），名为fastpool的 储存池 
zpool create -m /mnt/fastpool fastpool sdq sdx sdv
2.需要创建一个 2盘 硬盘编号sdx sdy ，镜像（类似raid1），名为testpool的 储存池 
zpool create -m /mnt/testpool testpool mirror sdx sdy
3.需要创建一个 6盘 硬盘编号sda sdb sdc sdd sde sdf，RAIDZ2(raidz2)，名为mypool的 储存池 
zpool create -m /mnt/mypool mypool raidz2 sda sdb sdc sdd sde sdf

5. 完成池的创建后，就可以使用命令：

zpool status [池名字]

查看池的具体状态
可以看到池的健康度，以及有没有报错
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%208.png)
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%209.png)
    
6. 

## 导入池

从TrueNAS转移后，直接导入原有储存池

---

1. 通过命令查看已经池
**zpool import**
可以看到已经查看到 **testpool** 储存池
    
    ![684B1490-639C-4A85-825B-8D8C28CD9F64.jpeg](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/684B1490-639C-4A85-825B-8D8C28CD9F64.jpeg)
    
2. 查找你需要的储存池后，就可以使用命令：
**zpool import [池名字]**
后，就可以使用命令：
**zpool status** 
查询池状态
    
    ![94188769-F218-4E26-BFB1-AA166AEAD1CF.jpeg](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/94188769-F218-4E26-BFB1-AA166AEAD1CF.jpeg)
    
3. 完成池导入

## Unraid 建立 Docker && VM（虚拟机）数据 并设置 路径 到对应目录

> 你需要在 存储池 下面 建立Docker 和 VM（虚拟机）的 数据集
很多人对数据集的概念比较陌生，简单的可以理解为一个可以有更多ZFS选项的文件夹
如果你是小白不需要任何改动，所以就当成文件夹使用。至于为什么不直接创建文件夹是，创建很简单，且当你熟悉ZFS后能有更多调整项。
> 

1. 建立 数据集
**zfs create [池名字]/[数据池]**
示例：
池的名字以刚刚的testpool为例子，建立名为docker以及vms的数据集，以下2条命令为：
zfs create testpool/docker
zfs create testpool/vms
即可完成数据集的建立
2. 使用下面命令可以查看ZFS池下面的所有数据集
**zfs list**
    
    ![756473BF-AA0A-45D7-90E6-A4F50AAF85C8.jpeg](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/756473BF-AA0A-45D7-90E6-A4F50AAF85C8.jpeg)
    
    就可以查看testpool下面的数据集的 
    名字(NAME) 占用(USED) 可用容量(AVAIL) 引用(REFER)  挂载点(MOUNTPOINT) 
    
3. 在UNRAID界面 设置 →  Docker/虚拟机管理 的2处 修改路径 到 刚刚创建的路由中
如图示例 ：     （PS：可能需要打开右上角的高级视图才能看见路径修改选项）
开头为
/mnt/[池名字]/[数据集]/[文件夹名]
注意这里的文件夹直接填，会自动创建，需要使用创建数据集的方式。
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%2010.png)
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%2011.png)
    

1. 全部设置完成，点击 应用  。
2. 此时因为Unraid一定需要启动一个阵列才能允许使用 Docker && VM（虚拟机）
3. 在 主界面 → 磁盘1 随便选择一块硬盘

    
    ![C18DFC3E-8033-4B48-A749-780951ED88AF.jpeg](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/C18DFC3E-8033-4B48-A749-780951ED88AF.jpeg)
    
    点击 启动 即可打开 Docker && VM（虚拟机）功能
    
    警告‼️：启动后**不要**点击 **格式化** ，因为我们只需要启动阵列来开启 对应功能，ZFS阵列已经存在。
    

## 挂载ZFS储存池 SMB共享

> 因为我们使用的是非unraid自带阵列，所以我们无法使用共享中的SMB共享
> 

1. 添加用户
在 用户 → 添加用户 中 设置自己的账户，用于SMB访问
2. 添加 SMB 设置
在 设置 → SMB → SMB额外 → Samba额外配置 中
    
    ![F8315DFE-07F4-4DE8-93B6-A0F3C447A980.jpeg](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/F8315DFE-07F4-4DE8-93B6-A0F3C447A980.jpeg)
    
    添加
    
    **[[随意名字]]
        path=/mnt/[池名字]
        valid users = [用户名]
        write list = [用户名]**
    
    示例： 添加池名字为 testpool 用户名为bob的SMB共享
    
    [testpool]
        path=/mnt/testpool
        valid users = bob
        write list = bob
    
    1. 点击应用，即可完成SMB共享
    
    需要注意的是，如果设置SMB共享时，需要停止Unraid系统自带阵列，即此处的停止
    
    ![1DCB475D-9D44-4D37-9EE3-3A36B3D8B87E.jpeg](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/1DCB475D-9D44-4D37-9EE3-3A36B3D8B87E.jpeg)
    
    完成设置后，需要重新启动阵列，就可以正常使用了。
    
    ## 设置ZFS内存缓存容量 && 定期校验储存盘
    
    - 设置ZFS内存缓存容量
    
    命令如下：
    
    `echo "echo 8589934592 >> /sys/module/zfs/parameters/zfs_arc_max" >> /boot/config/go`
    
    这里唯一需要调整的参数是 橙色 部分，这里代表单位是的 Bytes，转换为GB为8GB
    即设置ZFS缓存为8GB
    考虑到Docker和虚拟机运行也需要内存，需要根据你自己的需要调整大小，官方文档的建议是大于8GB，并设置不可大于本机最大容量，TrueNAS SCALE里面的默认数值是本机内存的一半。
    
    - 定期校验储存盘
    
    命令如下
    
    `4 2 4 * * zpool scrub [池名字] >> /dev/null`
    
    橙色 部分是**Linux** **Crontab**定时任务时间，需要自己查询调整需要校验的时间。
    
    青色 部分是池名字
    
    示例：  每个月1号 1:00AM 校验池 testpool
    
    1 0 1 * * zpool scrub testpool >> /dev/null
    
    ## 定时自动快照
    
    这里推荐使用Unraid ZFS插件同作者的ZnapZend插件
    
    ![498C52BE-25C5-498C-B7C5-EFEBEF409230.jpeg](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/498C52BE-25C5-498C-B7C5-EFEBEF409230.jpeg)
    
    可以在应用中直接安装
    
    1. 需要在命令行设置启动以及自动启动命令：
    `znapzend --logto=/var/log/znapzend.log --daemonize`
    `touch /boot/config/plugins/unRAID6-ZnapZend/auto_boot_on`
    2. 使用下面的命令，创建你需要多久创建一次快照的命令。
    `znapzendzetup create --recursive SRC '7d=>1h,30d=>4h,90d=>1d' SSD`
    橙色的代表 7天内每天保持24个备份，一个月内每天6个备份，然后在90天内每天进行一次快照。（看不懂建议照抄）
        
        青色的代表 池名字
        
    
    ### 查看快照
    
    **zfs list -t snapshot [池名字]/[数据集]**
    示例： 
    zfs list -t snapshot testpool/docker
    
    ### 回滚到快照
    
    **zfs rollback -r [快照名]**
    
    ## ZFS Master （一个有趣的插件）
    
    这是一个推荐的插件，可以在主菜单下面显示各个数据集的状态。
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%2012.png)
    
    ![Untitled](https://list.hiscloud.cc/d/SCALE2Unraid/Pic/Untitled%2013.png)
    

> 教程到这里就结束了，对于大部分的家用来说，ZFS池的默认选项能满足90%需求，但是ZFS池选项还有很多可以调整，后面也会出一期ZFS命令行的教程，来讲一下使用命令行调整池以及数据集选项。
> 

本文参考****ZFS PLUGIN FOR UNRAID****：[https://forums.unraid.net/topic/41333-zfs-plugin-for-unraid/](https://forums.unraid.net/topic/41333-zfs-plugin-for-unraid/)

@ChanningHe 转载需要标注
