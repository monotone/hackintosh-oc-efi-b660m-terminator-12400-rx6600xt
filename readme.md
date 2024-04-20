## 2024-04-20 更新

闲着没事又折腾了一波，直接从12.6升到了 14.4.1。目前用着还算完美，记录下踩了的坑。

![1713603632011](assets/readme/1713603632011.png)

#### 先禁用 secureBootMode

![1713604130427](assets/readme/1713604130427.png)

刚开始想着直接升级就行，于是直接走系统的更新流程，或者从appstore下载。结果弄好一个installer后重启，进入installer选项不能启动更新流程，但是老的macos还是能进去，原因不明，然后就一通折腾各种去更新 OC啊，找别人的 EFI 啊，通通都没搞定。

最后想着自己的efi既然能进去，理论上驱动应该大多数都是兼容的才对，所以又从自己的 EFI 出发，重新更新系统，不过这次我是先更新到 13，再更新到 14，并且使用的是下面的命令来下载完整的镜像(dmg结尾)，

```shell
mkdir -p ~/macOS-installer && cd ~/macOS-installer && curl https://raw.githubusercontent.com/munki/macadmin-scripts/main/installinstallmacos.py > installinstallmacos.py && sudo python installinstallmacos.py
```

我不确定直接从12升到 14行不行，但是可以尝试，反正原来的efi记得保留，然后有一个u盘能用来启动，就随便改。

#### Memory Modules Misconfigured 问题

装完之后发现会通知提示这个，关了之后实际也没啥影响。

oc官方介绍[Fixing MacPro7,1 Memory Errors | OpenCore Post-Install (dortania.github.io)](https://dortania.github.io/OpenCore-Post-Install/universal/memory.html#mapping-our-memory)，查了下大概就是指系统检测到的内存插槽没插满，至少包括 4 条这样性能比较好吧。（我刚好是只插了两条 8G的）

![1713604238771](assets/readme/1713604238771.png)

参照官方介绍，使用[acidanthera/RestrictEvents (github.com)](https://github.com/acidanthera/RestrictEvents)应该能解决，可以我加了之后还是有提示，看了readme里要添加启动参数，测试了之后还是不行。

![1713604423678](assets/readme/1713604423678.png)

最后参照网上的搞法自定义内存解决，就是加完重启后用oc打开看不到了，奇怪。估计是版本兼容性问题吧，暂时不管了。EFI 直接更新到最新版本了。

![1713606282672](assets/readme/1713606282672.png)

## 说明

本文仅记录最简化的步骤流程，因为发现前面折腾了好久都是多余的步骤，实际上参照cpu架构的说明文档简单靠谱。

https://github.com/luchina-gabriel/BASE-EFI-INTEL-DESKTOP-12THGEN-ALDER-LAKE

**目前我的一切正常，使用和白苹果无差异。(从 12.3 直接升级到了 12.6 无问题）**

![image-20221004143616388](./assets/image-20221004143616388.png)

所以如果下次再搞黑苹果，流程大概是：

1. 确定是全新组机还是在旧设备上做增量。建议都优先选用得多的。
2. 搜索对应cpu架构支持情况，如果搜不到就直接搜对应型号
3. 搜索对应主板的efi文件，基本能复用 ACPI 和 kexts
4. 规划好硬盘，一定要是 GPT 分区表

## 基本配置

cpu: intel 12400

主板：铭瑄 b660m 终结者

硬盘：WD Blue SN570 500GB

显卡：迪兰 AMD ADEON RX6600xt

OC 版本：0.8.4

无线网卡：**无（所以我没有打蓝牙和无线网卡的驱动）**

## 难点记录

1. 主板的cfg lock 关闭，因为BIOS 没有直接提供对应的操作入口，虽然直接在efi里设置也能启动，但是都推荐还是解锁掉。efi/tools/cfglock.efi就是用来关掉这个的，记得在misc/tools下开启这个efi。
2. USB 端口映射用到了好几个工具最后得到了一个kexts/usbports.kext，感觉还挺麻烦，可以考虑搜索同主板的 EFI 直接拷贝过来

## 更换硬盘

之前装双系统是在单个硬盘上，为了方便描述，直接说 0 号硬盘吧。只有 500G。大概分区形式是：

`FAT32(ESP) ：NTFS（windows C/D 盘）:  APFS (macOS) :exFAT(两个系统共享用)`

然后用久了macOS 空间不太够了。恰好现在的硬盘也便宜好多了。于是补了一块 2T 的 SSD。我们称其为 1 号硬盘。

因为macOS用的比较多，而且扩容的时候它只能向它后面的分区扩容，而且之前的用用法上exFAT 这个共享盘长期用来操作的路径也很奇怪不好搞，所以尽量给macOS多的空间，并且将来还有扩容的机会。所以我打算把macOS移到1号的第一个分区，这样后面的空间只要不存放重要数据，什么时候想扩容都方便。

```
0号硬盘（500G）： FAT32(ESP) ：NTFS（windows 500G）
1 号硬盘（2T）： FAT32(ESP）：APFS（macOS 600G）: APFS (预留空闲900G)：exFAT(public 500G用于两个系统间共享文件)
```

之前没太了解UEFI引导的启动方式，所以走了很多弯路搞了差不多一整天，这里记录下大概流程。

1. 在windows 下使用diskgenus(没拼错吧，反正随便一搜就搜的到)直接格式化 1 号硬盘，分配好 ESP 分区。（如果是把windows移到 1 号硬盘，可以直接用系统迁移功能）
2. 使用 DG 的分区克隆功能，把 0 号硬盘上的 APFS 分区拷贝到 1 号硬盘的第一个分区上。
3. 这时候不要忙着去删除 0 号硬盘上的分区，先把 0 号硬盘上的 ESP 分区里的 EFI 拷贝到 1 号的 ESP 分区里。
4. 然后重启一下，操作系统应该能识别到 1 号硬盘里的OpenCore启动，如果识别不到，可以在 DG 的工具里打开 UEFI编辑器，添加 1 号硬盘ESP 分区里的 `/EFI/OC/opencore.efi`作为启动项，然后重新启动的时候选择这个启动项（实际就是选择 OC 启动）
5. OC 启动后会自动扫描所有的macos系统分区，我这里应为 0 号合 1 号上各有一个分区有macOS系统，所以会有两个。
6. 因为是克隆的，所以两个盘符一模一样，我也分不清哪个是 1 号盘上的，就两个都启动了一下，发现都能启动。用起来也没啥问题似乎。但是打开disk util的时候，发现两个盘都被挂到了0号盘下。
7. 于是我担心会有问题，就做实验把0 号硬盘先取了，再开机发现用disk util看到正常被挂到 1 号硬盘下了。
8. 那就可以放心的删掉 0 号盘上的macOS了，剩下的就是用 SD 把 0 号盘上的分区做整合，这里就不说了。
9. 最后，清理0 号ESP 分区里的 OC，清理 1 号 ESP 分区里的Microsoft目录，这样就不会存在一些没用的引导项了（电脑启动的时候会去全硬盘扫描 ESP 分区，并把里面的efi启动项都给列出来似乎）
10. 补充一条，macOS出现了浏览器不能上网，别的工具可以的情况，不知道是不是迁移导致的，可以考虑手动配一下 DNS，并且清理下DNS 缓存：`sudo killall -HUP mDNSResponder`

有一个细节是，/EFI/BOOT/bootX64.efi这个文件，在没有主动增加启动项的时候默认会使用这个来启动，这里主要看后装哪个系统，后装的系统会用自己的efi覆盖这个，就成了默认启动系统。

可以忽略这个，手动添加启动项，macos的话就是/efi/oc/opencore.efi，windows的话就是/etfi/microsoft/boot/bootmgrwf.efi（不记得目录是不是这个了，反正长不多）。

还有windows 系统在发生移动之后，原来的 ESP 就起不来系统了（蓝屏，报xxxxx目录下的winload.efi找不到），原因应该是efi里的 bootmgrwf.efi里记录了对应的分区（GUID）和位置等信息，所以不能正常启动了。方法是想办法把安装这个windows的时候产生的bootmgrwf.efi复制过去替换掉。

疑问：

我记得有一种情况是选了类似 OC 的启动项后，能看到windows和macOS的启动硬盘，然后去选择，这种是怎么弄的呢? 下次遇到了仔细看看，有需要在记录吧。先这样。
