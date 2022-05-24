# SNMPTTCONVERTMIB

**SNMPTTCONVERTMIB**是一个 Perl 脚本，它将读取 MIB 文件并将**TRAP-TYPE** (v1) 或 **NOTIFICATION-TYPE** (v2) 定义转换为**SNMPTT**可读的配置文件。

例如，如果文件**CPQHOST.mib** (v1) 包含：

```bash
CPQHOST-MIB DEFINITIONS ::= BEGIN
    IMPORTS
        enterprises             FROM RFC1155-SMI
.
.
. (lines removed)
.
.
    cpqHo2NicSwitchoverOccurred2 TRAP-TYPE
        ENTERPRISE compaq
        VARIABLES { sysName, cpqHoTrapFlags, cpqHoIfPhysMapSlot,
                    cpqHoIfPhysMapPort, cpqHoIfPhysMapSlot,
                    cpqHoIfPhysMapPort }
        DESCRIPTION
            "This trap will be sent any time the configured redundant NIC
            becomes the active NIC."

             --#TYPE "Status Trap"
             --#SUMMARY "NIC switchover to slot %s, port %s from slot %s, port %s."
             --#ARGUMENTS {2, 3, 4, 5}
             --#SEVERITY MAJOR
             --#TIMEINDEX 99
        ::= 11010
```

执行 **snmpttconvertmib CPQHOST.mib snmptt.conf** 将附加以下信息到 **snmptt.conf** 文件的末尾（在命令行中指定）：

```bash
#
#
#
EVENT cpqHo2NicSwitchoverOccurred2 .1.3.6.1.4.1.232.0.11010 "Status Events" Normal
FORMAT Status Trap: NIC switchover to slot $3, port $4 from slot $5, port $6.
#EXEC qpage -f TRAP notifygroup1 "Status Trap: NIC switchover to slot $3, port $4 from slot $5, port $6."
SDESC
This trap will be sent any time the configured redundant NIC
becomes the active NIC.
EDESC
```

注意：

要指定 EXEC 语句，请使用 --exec=command 选项。 

要防止将 --#TYPE 文本附加到 --#SUMMARY 行，请在 **SNMPTTCONVERTMIB** 脚本中将 **$prepend_type** 更改为**0 。**

有关更多选项，请参阅帮助文档 ( **snmpttconvertmib --h** )。

## 需求条件

- UCD-SNMP/Net-SNMP [**snmptranslate**](http://www.net-snmp.org/man/snmptranslate.html) 程序
- **可选**：UCD-SNMP/Net-SNMP **[Perl 模块](http://www.net-snmp.org/FAQ.html#Where_can_I_get_the_perl_SNMP_package_)**

**snmpttconvertmib 使用 snmptranslate** 程序转换 MIB 文件。

如果在命令行上使用 **--net_snmp_perl** 启用了 Net-SNMP Perl 模块，它可以在 DESC 会话中提供更详细的变量描述（如果可用），例如：

- 变量语法
- 变量描述
- 变量枚举

例如：

```bash
2: globalStatus
   Syntax="INTEGER"
      2: ok
      4: failure
   Descr="Current status of the entire library system"
```

## 转换MIB文件

在转换 MIB 文件之前，请参阅 **snmpttconvertmib** 帮助文档以了解所有可能的命令行选项 ( **snmpttconvertmib --h )。**根据 MIB 文件中可用的信息类型，您可能希望更改 FORMAT / EXEC 行的生成方式。

在尝试转换 MIB 文件之前，您应该确保 MIB 文件可以被 Net-SNMP 解析

1. 将 MIB 文件复制到 UCD-SNMP / Net-SNMP mibs 文件夹
2. 类型：**export MIBS=ALL** 以确保所有 mib 都将被 **snmptranslate** 读取
3. 确保 MIB 文件可以被 **snmptranslate** 正确解释。只需键入 **snmptranslate** 即可告诉您它是否能够正确读取 mib 文件。如果不能，则会在帮助屏幕顶部产生错误。
4. 尝试翻译MIB 文件中包含的**TRAP-TYPE**或**NOTIFICATION-TYPE条目**。例如，如果 MIB 文件包含“ **rptrHealth NOTIFICATION-TYPE** ”的Notification定义，则键入： **snmptranslate rptrHealth -IR -Td**。如果您收到 **Unknown object identifier: xxx** ，则未找到或未正确解析 MIB 文件。

### 运行 snmpttconvertmib

1. 确保 MIB 文件已成功安装（见上文）

2. 如果需要，在 snmpttconvertmib 中编辑 OPTIONS START 和 OPTIONS END 之间的选项

3. 如果您使用的是**UCD-SNMP**或**Net-SNMP v5.0.1**，将以下内容添加到您的 snmp.conf 文件中：**printNumericOids 1**（注意：这将影响所有 snmp 命令）。这可确保 OID 以数字格式返回。其他版本的 Net-SNMP 不需要此更改，因为 **snmpttconvertmib** 在调用**snmptranslate** 时将使用命令行强制它打开。

4. 使用以下命令转换 mib 文件：**snmpttconvertmib --in= \*path-to-mib\* --out= \*output-file-name\* **。注意：**output-file-name**的动作为追加，因此请记住在需要时先将其删除。

   例子：

   ```bash
   snmpttconvertmib --in=/usr/share/snmp/mibs/CPQHOST.mib --out=/etc/snmp/snmptt.conf.compaqhost
   ```

   如果安装了 Net-SNMP Perl 模块并且您想要更多描述性的变量描述，请将 **--net_snmp_perl** 添加到命令行：

   ```bash
   snmpttconvertmib --in=/usr/share/snmp/mibs/CPQHOST.mib --out=/etc/snmp/snmptt.cong.compaq --net_snmp_perl
   ```

   要转换当前文件夹中的所有 CPQ* 文件，您可以使用：

   Unix / Linux：

   ```bash
   for i in CPQ*
   > do
   > /usr/local/sbin/snmpttconvertmib --in=$i --out=snmptt.conf.compaq
   > done
   ```

   Windows:

   ```bash
   for %i in (CPQ*) do perl snmpttconvertmib --in=%i --out=snmptt.conf.compaq
   ```

## 工作原理

 一些 MIB 文件包含 Novell 的网络管理系统使用的 **--#SUMMARY** 和 **--#ARGUMENTS** 行。这些 MIB 文件可以很好地转换为**SNMPTT**，因为它们包含可用于**FORMAT**和 **EXEC**行的详细信息。Compaq 的 MIB 通常有这些行。

其他 MIBS 仅包含一个 **DESCRIPTION** 部分，其中第一行包含 **FORMAT** 字符串。在某些 MIBS 中，此行还包含类似于 **--#SUMMARY** 行的变量。

在 mib 文件中搜索 MIB 文件的名称。它位于文件的顶部并包含'**name DEFINITIONS ::=BEGIN**'。此名称将在查找 TRAP/NOTIFICATION 时使用，以确保访问正确的 MIB 文件。

还会在 mib 文件中搜索包含 **TRAP-TYPE**或 NOTIFICATION-TYPE 的行。在**DESCRIPTION** 部分如果它找到一个似乎是有效的陷阱定义， 它会读取紧接着的行，直到找到 ::=。然后它会查找 **--#SUMMARY** 和 **--#ARGUMENTS** 行（如果启用的话）。

**SNMPTRANSLATE** 使用以下语法查找陷阱的**OID**：

```bash
snmptranslate -IR -Ts mib-name::trapname -m mib-filename
```

注意：如果检测到 Net-SNMP 5.0.2 或更高版本，命令行还要包括 -On 开关。请参阅 [常见问题解答](file:///H:/cvs/snmptt/readme.html#FAQ-Troubleshooting)。

- 如果找到 **--#SUMMARY** 和 **--#ARGUMENTS**，则 **%*letter*** 变量将根据 **--#ARGUMENTS** 部分中的值列表以步长为1递增（ARGUMENTS 以0开始，SNMPTT 以1开始） 替换为 **$*number*** 变量. 这将用于定义**FORMAT**和**EXEC**行。
- 如果没有 **--#SUMMARY** 和 **--#ARGUMENTS** 行，但 **DESCRIPTION** 的第一行包含 **%*letter*** 变量，则该行将用于定义 **FORMAT** 和 **EXEC **行。**%*letter*** 变量被替换为从 1 开始向上递增的 **$*number*** 变量。
- 如果没有 **--#SUMMARY** 和 **--#ARGUMENTS** 行，并且**DESCRIPTION**的第一行不包含 **%*letter*** 变量，则该行将被 sed 处理以定义**FORMAT** 和**EXEC**行，然后是 **$\*** 转储所有收到的变量。
- 如果条目包含变量，则变量会在 DESC 部分中列出。如果指定了 **--net_snmp_perl**，则使用每个变量的语法、描述和枚举。
- 注意：这可以通过指定 --format=n 命令行选项来更改。有关所有可能的命令行选项 (snmpttconvertmib --h)，请参阅 snmpttconvertmib 帮助文档。

