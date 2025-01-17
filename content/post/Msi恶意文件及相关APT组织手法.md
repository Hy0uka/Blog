+++
title = 'Msi恶意文件及相关APT组织手法'

description = "虽然不是很厉害但至少开始写了"

tags = [ "APT" ]

date = 2024-11-12T11:15:13+08:00

+++

**Msi文件**：即microsoft installer的简写，msi文件就是window installer的**数据包**，把所有和安装文件相关（即下文说的windows installer的功能）的内容封装在一个包里。是包含有关软件安装的详细信息的数据库文件，存储安装过程中的数据，包括配置设置、要安装的文件和安装说明，确保结构化和高效的软件部署。

**Windows Installer的用途包括**:管理软件的安装、管理软件组件的添加和删除、监视文件的复原以及使用回滚技术维护基本的灾难恢复。



**Msi技术的原理**：Windows Installer技术就是合并在一起发挥作用的两个部分:客户端安装程序服务（Msiexec.exe） 和Microsoft软件安装（MSI）软件包文件。

Msiexec.exe 程序是 Windows Installer 的一个组件。 当 Msiexec.exe 被安装程序调用时，它将用 Msi.dll 读取**软件包文件(.msi)、应用转换文件(.mst)** 并合并由安装程序提供的命令行选项。 Windows Installer 执行所有与安装有关的任务：包括将文件复制到硬盘、修改注册表、创建桌面快捷方式、必要时显示提示对话框以便用户输入安装首选项。

当双击MSI文件的时候，与之关联的Windows Installer 的一个文件Msiexec.exe 被调用，它将用Msi.dll读取软件包文件（.msi）、应用转换文件（.mst）进行进一步处理。



MSI文件的结构是Windows Installer用于软件安装、维护和删除的**关系数据库**，数据库由几个表组成，每个表在安装过程中有特定的用途，这些表通过主键值和外键值链接。安装数据库包含安装应用程序所需的所有信息，因此需要用所需的信息填充表。一个MSI文件可以有多个表，我们重点关注在我们的分析中最有用的表。

**二进制表**直接在MSI数据库中存储二进制数据，如图标、位图、动画或自定义操作脚本，这些数据直接嵌入MSI文件中，并在安装期间使用Windows Installer API进行访问，它通常用于安装过程本身所需的较小资源。

文件表包含有关将安装在目标系统上的文件的信息，描述要复制到用户计算机的实际应用程序文件。这些文件通常单独存储（例如，在CAB文件中）或作为MSI中的压缩流，并表示构成正在安装的应用程序的文件，文件表用于确定要安装的文件、安装位置及其属性。



**Custom Action表**

MSI文件中的自定义操作是指在安装过程中运行的命令或脚本，用于执行标准MSI操作无法处理的自定义任务。这些操作可以执行可执行文件、DLL、VBScript、JavaScript或预定义命令，从而使攻击者能够在看似合法安装的后台执行恶意代码

MSI文件中的自定义操作在CustomAction表中定义，此表中的每个条目指定一个自定义操作、其类型、源和目标。以下是用于定义自定义操作的列的细分：

- Action：这是自定义操作的名称。它充当主键并在序列表中引用此操作。如果名称与任何内置操作匹配，则不会执行自定义操作。
- Type：此字段包含指定自定义操作类型及其执行选项的标志的按位组合。类型确定操作是调用DLL、可执行文件、脚本还是设置属性。
- Source：此字段可以包含属性名称或指向另一个表的键，例如Binary表（用于嵌入的二进制文件）、File表（用于已安装的文件）或Directory表（用于路径）。引用的特定表取决于自定义操作的类型。
- Target：此字段提供自定义操作所需的其他信息，如DLL的入口点、可执行文件的命令行参数或脚本代码。
- ExtendedType：此字段指定自定义操作的其他选项，例如处理补丁卸载。

这里就可以完成很多预期之外的操作，例如运行可执行文件的自定义操作可能使其Source字段引用File表中的键，而其Target字段包含命令行参数。

**常见文件扩展名及其作用**

- .msi：Microsoft Windows软件包文件的主要扩展名，包含安装数据库和安装软件的必要说明。
- .mst：Windows NT转换文件，用于修改或自定义MSI文件，而不改变原始软件包。
- .msm：合并模块文件，用于交付共享组件并确保跨多个应用程序安装相同版本的共享文件。
- .msp：Windows Installer修补程序文件，用于对已安装的应用程序应用更新或修复。
- .cab：Cabinet文件，用于将多个文件压缩和打包到一个文件中，以便于分发。CAB 文件可以显著减小安装包的大小，并将多个文件打包在一起，以简化分发和部署。

**手动分析MSI文件的工具**

- msitools：包括msiinfo和msidump等实用程序，可用于检查MSI文件，枚举表和流，转储表内容，并从MSI文件中提取数据。
- msidump：由Mgeeky开发的工具，用于分析恶意MSI软件包，可以提取文件、流和二进制数据，并集成了YARA扫描，对于快速分类和详细检查潜在的恶意MSI特别有用。
- lessmsi：一个具有图形用户界面和命令行界面的实用程序，可以用来查看和提取MSI文件的内容，还具有Windows资源管理器集成，便于提取和MSI表查看器。
- MSI Viewer：可从Microsoft Store获得，用以查看MSI安装程序文件和合并模块的内容，还可以提取文件，而无需运行安装程序。
- Orca：由Microsoft开发的图形用户界面（GUI）工具，用于分析和编辑MSI文件，Orca允许用户打开MSI文件并查看其内部结构，包括表，属性和安装包的其他组件。
- msidiff：可以用于比较两个msi文件的差异。



**活跃APT组织使用MSI作为相关攻击载体情况**

Bitter、APT-Q-27、APT-Q-15(Darkhotel)、CNC等APT组织将恶意组件压缩在cab中，在msi安装过程中释放并执行，这也是目前最为常见的利用手法，缺点是恶意组件随着MSI的安装会落地在磁盘上，比较考验攻击者持续的免杀技术。



在CustomAction中支持各种类型的自定义操作，攻击者有较为丰富的操作空间，例如Bitter组织在CustomAction表中调用带有签名的第三方powershell解释器执行powershell脚本。



而APT-Q-15（Darkhotel）在针对朝鲜人的间谍活动中，投递恶意的朝鲜字体MSI安装包，将木马模块core.dll添加到customAction表内，与Media表中插入的恶意模块相比，core.dll在msi安装过程中并不会落地，系统进程msiexec会启动一个独立的子进程内存加载该DLL，从而达到LOLBINS的效果。



**MST文件和MSI文件的关系**

**MST (Microsoft Transform) 文件**是一种变换文件，它用于自定义和修改 MSI 文件的安装过程。通过使用 MST 文件，管理员可以对 MSI 安装包进行定制化调整，而不需要直接修改原始的 MSI 文件。这种方式非常适用于批量部署，因为可以根据不同需求灵活调整安装设置。

**使用方法**

1. **生成 MST 文件**：
   - 可以使用工具如 InstallShield、Orca 等来创建和编辑 MST 文件。
2. **应用 MST 文件**：
   - 在安装 MSI 文件时，可以使用命令行参数将 MST 文件应用于 MSI 包。例如：

```
     msiexec /i example.msi TRANSFORMS=example.mst
```

- 这条命令会安装 example.msi 文件，同时应用 example.mst 文件中的配置。

所以其实只是LOLbins的一种实现方式，不过“黑”从dll变成了mst。mst可以使用ocra生成，

MST文件实际上也是压缩包，解压开来分析其中的dll就行。

MST 内部的可执行模块一般会有两个导出函数分别为 LogSetupAfterInstall 和 LogSetupBeforeInstall，用来控制 MSI 安装过程中的流程。



*参考：*

[MSI文件滥用新趋势：新海莲花组织首度利用MST文件投递特马](https://ti.qianxin.com/blog/articles/new%20-trend-in-msi-file-abuse-new-oceanlotus-group-first-to-use-mst-files-to-deliver-special-trojan-cn/)

[msi恶意软件分析](https://bbs.kanxue.com/thread-258044.htm )

[MSI文件结构详解及MSI恶意样本分析方法](https://xz.aliyun.com/t/16134?time__1311=GuD%3D7IqGhx%2FnKYK0%3DvBxmTr%3DwDOimeD)

[MSI Shenanigans. Part 1 – Offensive Capabilities Overview](https://mgeeky.tech/msi-shenanigans-part-1/)

[How to Analyze Malicious MSI Installer Files](https://intezer.com/blog/incident-response/how-to-analyze-malicious-msi-installer-files/)
