# SNMPTT简介

SNMPTT (SNMP Trap Translator) 是一个用 Perl 编写的 SNMP trap 处理程序，用于 Net-SNMP/UCD-SNMP [snmptrapd](http://www.net-snmp.org/man/snmptrapd.html) 程序 ( [www.net-snmp.org](http://www.net-snmp.org/) )。SNMPTT 支持 Linux、Unix 和 Windows。

许多网络设备，包括但不限于网络交换机、路由器、远程访问服务器、UPS、打印机和如 Unix 和 Windows NT的操作系统，都具有向运行在 NMS 上的 SNMP 管理器发送通知的能力。通知可以是 SNMP trap 或 SNMP inform 消息。

通知可以包含大量信息，例如端口故障、链接故障、违规访问、断电、卡纸、硬盘驱动器故障等。供应商提供的 MIB（管理信息库）决定了每个设备支持的通知。

MIB 文件包含 TRAP-TYPE (SMIv1) 或 NOTIFICATION-TYPE (SMIv2) 的定义，它们定义发生特定事件时传递给 NMS 的变量。

Net-SNMP 程序的 **snmptrapd** 进程是一个通过 TCP/IP 接收和记录 SNMP trap和 inform 消息的应用程序。以下是Compa 的 qcpqDa3LogDrvStatusChange trap的系统日志示例条目，该trap通知驱动器阵列正在重建：

```bash
Feb 12 13:37:10 server11 snmptrapd[25409]: 192.168.110.192: Enterprise Specific Trap (3008) Uptime: 306 days, 23:13:24.29, .1.3.6.1.2.1.1.5.0 = SERVER08, .1.3.6.1.4.1.232.11.2.11.1.0 = 0, .1.3.6.1.4.1.232.3.2.3.1.1.4.8.1 = rebuilding(7)
或者
Feb 12 13:37:10 server11 snmptrapd[25409]: 192.168.110.192: Enterprise Specific Trap (3008) Uptime: 306 days, 23:13:24.29, sysName.0 = SERVER08, cpqHoTrapFlags.0 = 0, cpqDaLogDrvStatus.8.1 = rebuilding(7)
```

snmptrapd 的输出可以通过 -O 选项更改为显示数字或 OID 和其他显示选项，但它通常遵循如下格式：变量名 = 值、变量名 = 值等。

使用 SNMPTT 的变量替换可以创建更具描述性/友好的trap消息。以下是使用 SNMPTT 记录的同一个trap：

```bash
Feb 12 13:37:13 server11 TRAPD: .1.3.6.1.4.1.232.0.3008 Normal "XLOGONLY" server08 - Logical Drive Status Change: Status is now rebuilding
```

SNMPTT 配置文件中 cpqDa3LogDrvStatusChange trap的定义如下：

```yaml
FORMAT Logical Drive Status Change: Status is now $3.
```

$3 代表 MIB 文件中定义的第三个变量，对于这个特定的trap，它是 cpqDaLogDrvStatus 的变量。

SNMPTT 配置条目的另一个示例如下：

```yaml
FORMAT Compaq Drive Array Spare Drive on controller $4, bus $5, bay $6 status is $3.
```

这可以得到以下输出：

```bash
Compaq Drive Array Spare Drive on controller 3, bus 0, bay 3 status is Failed.
```

Snmptt 的输出可以记录到以下任何目标：文本日志、syslog、NT 事件日志 或 SQL 数据库。也可以运行外部程序将翻译后的trap传递给电子邮件客户端、寻呼软件、Nagios 等。

除了变量替换，SNMPTT 还允许复杂的配置，允许：

- 接受或拒绝基于主机名、IP 地址、网络范围或 trap 企业变量内的变量值 的 trap 能力
- 执行外部程序以发送页面、电子邮件等
- 对翻译后的消息进行正则表达式搜索和替换，例如将变量值 “Building alarm 4” 翻译为 “Moisture detection alarm”



# 安装

## 概述

下面概述了安装和配置 SNMPTT 所需的一般步骤：

1. 按如下所述安装 Net-SNMP 和 SNMPTT
2. 手动创建 **snmptt.conf** 文件，或使用 snmpttconvertmib
3. 修改 snmptt.ini 以包含 **snmptt.conf** 文件并设置任何所需的选项
4. 启动 snmptt
5. 配置您的网络设备以将 trap 发送到 Net-SNMP / SNMPTT 设备上
6. 在您的网络设备上启动 trap 并检查 SNMPTT 日志文件以获取结果
7. 保护 SNMPTT

## Unix

#### 标准处理程序

标准处理程序是一个小型 Perl 程序，当使用守护进程模式时，每次 snmptrapd 接收到 trap 时都会调用该程序。此处理程序的限制是：

- 每次收到trap时，都必须创建一个进程来运行 snmptthandler 程序，并且每次都会读取 snmptt.ini。
- SNMPv3 的 EngineID 和名称不会通过 snmptrapd 传递给 snmptthandler。

使用这个处理程序的好处是：

- snmptrapd 不需要嵌入式 Perl。
- snmptt 中自 v0.1 以来一直存在。
- 足以满足大多数安装。

安装步骤：

1. 阅读整个文件以了解 snmptt 的工作原理

2. 将 **snmptt** 复制到**/usr/sbin/**并确保它是可执行的（**chmod +x snmptt**）

3. 将**snmptthandler**复制到**/usr/sbin/**并确保它是可执行的（**chmod +x snmptthandler**）

4. 将**snmptt.ini**复制到**/etc/snmp/**或 **/etc/**并编辑文件内的选项。

5. 将**examples/snmpt.conf.generic**复制到 **/etc/snmp/snmptt.conf** （在复制过程中重命名文件）或使用touch 命令创建文件（**touch /etc/snmp/snmptt.conf）。**

6. 创建日志文件夹**/var/log/snmptt/**。

7. 对于**独立模式：** 

   通过添加以下行来修改（或创建）Net-SNMP 的 **snmptrapd.conf文件：**

   ```bash
   traphandle default /usr/sbin/snmptt
   ```

   > 注意：可以将 snmptrapd 配置为根据接收到的特定trap执行**snmptt** ，但首选推荐使用 **default** 选项

8. 对于**守护进程**模式：

   a. 通过添加以下行来修改（或创建）Net-SNMP 的 snmptrapd.conf 文件：

   ```bash
   traphandle default /usr/sbin/snmptthandler
   ```

   b. 创建 spool 文件夹**/var/spool/snmptt/**：

   ```bash
   mkdir /var/spool/snmptt/
   ```

   c. 将脚本复制到 init.d 目录（在复制期间重命名文件）。该脚本是一个启动脚本，可用于在 Mandrake、RedHat 和其他系统上启动和停止 snmptt 。：

   ```bash
   cp snmptt-init.d /etc/rc.d/init.d/snmptt
   ```

   d.使用**chkconfig**添加服务：

   ```bash
   chkconfig --add snmptt
   ```

   e.将服务配置为在运行级别 2345 启动：

   ```bash
   chkconfig --level 2345 snmptt on
   ```

   f.snmptt 将在下次重新启动时启动，或者可以立即启动：

   ```bash
   service snmptt start
   或者
   /etc/rc.d/init.d/snmptt start
   ```

   g.要手动启动 snmptt，请使用：

   ```bash
   snmptt --daemon
   ```

9. 将文件复制到 logrotate.d 目录（在复制过程中重命名文件）。该脚本是日志轮换脚本，可用于轮换Mandrake、RedHat等系统上的日志文件：

   ```bash
   cp snmptt.logrotate /etc/logrotate.d/snmptt
   ```

   编辑 **/etc/logrotate.d/snmptt**并根据需要更新路径和轮换频率。

10. 使用命令行启动**snmptrapd ：** `snmptrapd -On`。

    您应该能够编辑 **/etc/rc.d/init.d/snmptrapd** 脚本（如果有的话）并将 OPTIONS 更改为**"** -On **"**。

    注意：建议使用 `-On`。这将使 snmptrapd 以数字形式传递 OID，并防止 SNMPTT 必须将符号名称转换为数字形式。如果没有安装 UCD-SNMP/Net-SNMP 的Perl 模块，那么您必须使用 -On 开关。根据 UCD-SNMP/Net-SNMP 的版本，某些符号名称可能无法正确转换。有关更多信息，请参阅常见问题解答。

    作为替代方案，您可以编辑 **snmp.conf** 文件以包含以下行：**printNumericOids 1。** 无论在命令行上使用什么，此设置都会生效。

11. 请参阅[保护 SNMPTT](http://snmptt.sourceforge.net/docs/snmptt.shtml#SecuringSNMPTT)部分。

    注意：默认的 snmptt.ini 启用日志记录到 snmptt.log，以及trap消息和 snmptt 系统消息的发送到 syslog。如果需要，可以更改以下部分的设置：`log_enable`、`syslog_enable` 和 `syslog_system_enable`。

#### 嵌入式处理程序

嵌入式处理程序是一个小型 Perl 程序，在 snmptrapd 启动时直接加载到 snmptrapd 中。此处理程序的限制是：

- snmptrapd 需要开启嵌入式 Perl
- 仅适用于守护进程模式

使用这个处理程序的好处是：

- 处理程序在 snmptrapd 启动时被加载和初始化，因此不需要创建新进程并且只进行一次初始化（加载 snmptt.ini），因此开销较少。
- SNMPv3 的 EngineID 和名称变量在 snmptt 中可用（变量 B*）

安装步骤：

1. 阅读整个文件以了解 snmptt 的工作原理

2. 确保 snmptrapd 已启用嵌入式 Perl 支持。在源代码编译时，使用`--enable-embedded-perl`配置选项。  

   输入 **snmptrapd -H 2>&1 | grep perl**。如果启用了嵌入式 Perl  ，它应该给出消息： **perl PERLCODE 。**

   如果它不可用，您需要使用`--enable-embedded-perl`配置选项编译 Net-SNMP。

3. 将**snmptt**复制到**/usr/sbin/**并确保它是可执行的（**chmod +x snmptt**）

4. 将**snmptthandler-embedded**复制到**/usr/sbin/**。它不需要是可执行的，因为它由 snmptrapd 直接调用。

5. 将**snmptt.ini**复制到**/etc/snmp/**或 **/etc/**并编辑文件内的选项。

6. 将**examples/snmpt.conf.generic**复制到 **/etc/snmp/snmptt.conf** （在复制期间重命名文件）或使用touch 命令创建文件（**touch /etc/snmp/snmptt.conf）。**

7. 创建日志文件夹**/var/log/snmptt/**。

8. 配置 snmptrapd 并安装服务：

   通过添加以下行来修改（或创建）Net-SNMP 的 snmptrapd.conf 文件：

   ```conf
   perl do "/usr/sbin/snmpthandler-embedded";
   ```

   创建 spool 文件夹**/var/spool/snmptt/**：

   ```bash
   mkdir /var/spool/snmptt/
   ```

   将脚本复制到 init.d 目录（在复制期间重命名文件）。该脚本是一个启动脚本，可用于在 Mandrake、RedHat 和其他系统上启动和停止 snmptt 。

   ```bash
   cp snmptt-init.d /etc/rc.d/init.d/snmptt
   ```

   使用**chkconfig**添加服务：

   ```bash
   chkconfig --add snmptt
   ```

   将服务配置为在运行级别 2345 启动：

   ```bash
   chkconfig --level 2345 snmptt on
   ```

   snmptt 将在下次重新启动时启动，或者可以立即启动：

   ```bash
   service snmptt start
   或者
   /etc/rc.d/init.d/snmptt start
   ```

   要手动启动 snmptt，请使用：

   ```bash
   snmptt --daemon
   ```

9. 将文件复制到 logrotate.d 目录（在复制过程中重命名文件）。该脚本是日志轮换脚本，可用于轮换Mandrake、RedHat等系统上的日志文件：

   ```bash
   cp snmptt.logrotate /etc/logrotate.d/snmptt
   ```

   编辑 **/etc/logrotate.d/snmptt**并根据需要更新路径和轮换频率。

10. 使用命令行启动**snmptrapd ：** `snmptrapd -On`。

    您应该能够编辑 **/etc/rc.d/init.d/snmptrapd** 脚本（如果有的话）并将 OPTIONS 更改为**"** -On **"**。

    注意：建议使用 `-On`。这将使 snmptrapd 以数字形式传递 OID，并防止 SNMPTT 必须将符号名称转换为数字形式。如果没有安装 UCD-SNMP/Net-SNMP 的Perl 模块，那么您必须使用 -On 开关。根据 UCD-SNMP/Net-SNMP 的版本，某些符号名称可能无法正确转换。有关更多信息，请参阅常见问题解答。

    作为替代方案，您可以编辑 **snmp.conf** 文件以包含以下行：**printNumericOids 1。** 无论在命令行上使用什么，此设置都会生效。

11. 请参阅保护 SNMPTT 部分。

    注意：默认的 snmptt.ini 启用日志记录到 snmptt.log，以及trap消息和 snmptt 系统消息的发送到 syslog。如果需要，可以更改以下部分的设置：`log_enable`、`syslog_enable` 和 `syslog_system_enable`。

## Windows

## 保护SNMPTT

与大多数软件一样，SNMPTT 应该在没有 root 或管理员权限的情况下运行。使用非特权帐户运行有助于限制使用**EXEC**和 **REGEX**等功能时可能发生的操作。

对于 Linux 和 Unix，如果您以 root 身份启动 SNMPTT，则应该创建一个名为“ **snmptt **”的用户，并且应该将**snmptt.ini** 的选项 `daemon_uid`设置为数字用户 ID（例如：500）或文本用户 ID（snmptt）。 **只有在使用 root 启动 snmptt 时才定义 daemon_uid。**

如果您以非 root 用户身份启动 SNMPTT，则不需要 `daemon_uid`（并且可能无法正常工作）。

在守护进程模式下使用`daemon_uid`时，将有两个 SNMPTT 进程。第一个将作为 root 运行并负责创建 .pid 文件，并在退出时清理 .pid 文件。第二个进程将以**daemon_uid**定义的用户身份运行。如果启用了系统的 syslog ( **syslog_system_enable** )，则会记录一条消息，说明用户 ID 已更改。从那时起的所有处理都将作为新的用户 ID。这可以通过检查 syslog 中的用户 ID 以获取trap和系统消息来验证。例如，如果用户 ID 更改为 500，则系统日志将包含带有**snmptt[500]**的条目。以 root 身份运行时，条目将包含**snmptt[0]**。

对于 Windows，称为 '**snmptt** ' 的本地或域用户帐户应该被创建。如果作为 NT 服务运行，则该服务应配置为使用**snmptt**用户帐户。否则，在以守护进程模式启动 SNMPTT 之前，应使用**snmptt**帐户在本地登录系统。 

从 Net-SNMP 的 **snmptrapd** 调用的脚本 **snmptthandler** 将在与**snmptrapd**相同的安全上下文中执行。

`snmptt`用户应配置以下权限：

- 对spool目录的 读取/删除 访问权限，以便能够读取新的trap，并删除已处理的trap
- 对配置文件（snmptt.ini 和所有 snmptt.conf 文件）的读取权限
- 对日志文件夹 /var/log/snmptt/ 或 c:\snmp\log\ 的写入权限。
- 执行 EXEC 语句所需的任何其他权限

如果**snmptrapd**以非 root/管理员身份运行，则应配置以下权限：

- 对spool目录的写访问权限

注意：建议只允许运行 **snmptrapd** 的用户和 **snmptt** 用户访问 spool 文件夹。这将防止其他用户将文件放入spool文件夹，例如非trap相关文件或导致 SNMPTT 重新加载 的非重新加载文件。

# snmptt.ini的配置选项

正如本文档中所提到的，配置选项是通过编辑**snmptt.ini**文件来设置的。

对于 Linux / Unix，搜索以下目录可以找到**snmptt.ini**：

```bash
/etc/snmp/
/etc/
/usr/local/etc/snmp/
/usr/local/etc/
```

对于 Windows，该文件应位于 **%SystemRoot%\\** 中。例如，**c:\winnt** 或 **c:\windows**。

可以使用 `-ini= 参数` 在命令行上设置 ini 文件的位置。请参阅[命令行参数](http://snmptt.sourceforge.net/docs/snmptt.shtml#Command-line-arguments)。

此软件包中提供了一个示例**snmptt.ini**。对于 Windows NT，请务必将 **snmptt.ini-nt** 文件复制到 **%SystemRoot%\snmptt.ini**。请务必从文件名的末尾删除 **-nt**。

该自述文件未记录 snmptt.ini 中可用的所有配置选项，因为 **snmptt.ini** 文件包含了每个选项的详细说明。

# Logging

## 标准输出

翻译后的trap可以发送到标准输出和日志文件。输出格式为：

```bash
date trap-oid severity category hostname translated-trap
```

要配置标准输出或常规日志记录，请编辑 **snmptt.ini**文件并修改以下变量：

```bash
stdout_enable
log_enable
log_file
```

## 未知traps

也可以记录未识别的trap。这将主要用于故障排除目的。

要配置未知trap日志记录，请编辑 snmptt.ini 文件并修改以下变量：

```bash
enable_unknown_trap_log
unknown_trap_log_file
```

如数据库部分 所述，未知trap也可以记录到 SQL 表中 。

## syslog

翻译后的trap也可以发送到 syslog。条目的格式与上面的类似，区别是没有日期（因为 syslogd 记录日期）：

```bash
trap-oid severity category hostname translated-trap
```

Syslog 条目通常以 **date hostname snmptt[ \*pid\* ]** 开头

要配置 syslog，编辑 snmptt ini 文件并修改以下变量：

```bash
syslog_enable
syslog_facility
syslog_level
```

通过编辑 snmptt.ini 文件并修改以下变量，可以将 SNMPTT 的系统错误发送到 syslog：

```bash
syslog_system_enable
syslog_system_facility
syslog_system_level
```

Syslog 系统条目通常以 **date hostname snmptt-sys[ \*pid\* ]** 开头：

以下错误会被记录：

​	SNMPTT (version) started **(\*)**
​	Unable to enter spool dir *x* **(\*)**
​	Unable to open spool dir *x* **(\*)**
​	Unable to read spool dir *x* **(\*)**
​	Could not open trap file *x* **(\*)**
​	Unable to delete trap file *x* from spool dir **(\*)**
​	Unable to delete !reload file spool dir **(\*)**
​	Unable to delete !statistics file spool dir **(\*)**
​	Reloading configuration file(s) **(\*)**
​	SNMPTT (version) shutdown **(\*)**
​	Loading *snmpttconfigfile* **(\*)**
​	Could not open configuration file: *snmpttconfigfile***(\*)**
​	Finished loading *x* lines from *snmpttconfigfile* **(\*)**
​	MySQL error: Unable to connect to database
​	SQL error: Unable to connect to DSN
​	Can not open log file *logfile*
​	MySQL error: Unable to perform PREPARE
​	MySQL error: Unable to perform INSERT INTO (EXECUTE)
​	DBI DBD::ODBC error: Unable to perform INSERT INTO
​	Win32::ODBC error: Unable to perform INSERT INTO
​	PostgreSQL error: Unable to connect to database
​	PostgreSQL error: Unable to perform PREPARE
​	PostgreSQL error: Unable to perform INSERT INTO (EXECUTE)
​	*** (仅daemon模式)**

## Eventlog

翻译的trap也可以发送到 NT 事件日志。所有trap都记录在源 **SNMPTT** 下的 **EventID 2** 中。条目的格式与上面的类似，区别是没有日期（因为事件日志记录日期）：

```bash
trap-oid severity category hostname translated-trap
```

要配置事件日志，请编辑 snmptt ini 文件并修改以下变量：

```bash
eventlog_enable
eventlog_type
```

通过编辑 snmptt.ini 文件并修改以下变量，可以将 SNMPTT 系统错误发送到事件日志：

```bash
eventlog_system_enable
```

以下错误会被记录。请注意，每个错误都包含一个唯一的**EventID**：

​	EventID 0: SNMPTT (version) started **(\*)**
​	EventID 3: Unable to enter spool dir *x* **(\*)**
​	EventID 4: Unable to open spool dir *x* **(\*)**
​	EventID 5: Unable to read spool dir *x* **(\*)**
​	EventID 6: Could not open trap file *x* **(\*)**
​	EventID 7: Unable to delete trap file *x* from spool dir **(\*)**
​	EventID 20: Unable to delete !reload file spool dir **(\*)**
​	EventID 21: Unable to delete !statistics file spool dir **(\*)**
​	EventID 8: Reloading configuration file(s) **(\*)**
​	EventID 1: SNMPTT (version) shutdown **(\*)**
​	EventID 9: Loading *snmpttconfigfile* **(\*)**
​	EventID 10: Could not open configuration file: *snmpttconfigfile***(\*)**
​	EventID 11: Finished loading *x* lines from *snmpttconfigfile***(\*)**
​	EventID 12: MySQL error: Unable to connect to database
​	EventID 13: SQL error: Unable to connect to DSN *dsn*
​	EventID 14: Can not open log file *logfile*
​	EventID 23: MySQL error: Unable to perform PREPARE
​	EventID 15: MySQL error: Unable to perform INSERT INTO (EXECUTE)
​	EventID 16: DBI DBD::ODBC error: Unable to perform INSERT INTO
​	EventID 17: Win32::ODBC error: Unable to perform INSERT INTO
​	EventID 18: PostgreSQL error: Unable to connect to database
​	EventID 22: PostgreSQL error: Unable to perform PREPARE
​	EventID 19: PostgreSQL error: Unable to perform INSERT INTO (EXECUTE
​	*** (仅daemon模式)**)

注意：要防止事件查看器中出现“Event Message Not Found”消息，必须使用事件消息文件。有关安装消息文件的信息，请参阅[Windows 的安装部分](http://snmptt.sourceforge.net/docs/snmptt.shtml#Installation - Windows)。

## 数据库

翻译的和无法识别的trap也可以发送到数据库。可以使用 MySQL（在 Linux 下测试）、PostgreSQL（在 Linux 下测试）和 ODBC（在 Windows NT 下测试）。

要配置未知trap日志记录，请编辑 snmptt.ini 文件并修改以下变量：

```bash
enable_unknown_trap_log
db_unknown_trap_format
```

### MySQL

### PostgreSQL

### ODBC

### Windows ODBC

## 调用一个外部程序

收到trap时可以启动外部程序。命令行在配置文件中定义。例如，要使用 QPAGE ( [http://www.qpage.org](http://www.qpage.org/) ) 发送页面，可以使用以下命令行：

```bash
qpage -f TRAP notifygroup1 "$r $x $X Compaq Drive Array Spare Drive on controller $4, bus $5, bay $6 status is $3."
```

$r 为主机名，$x 为当前日期，$X 为当前时间（下面详细介绍）

要启用或禁用 EXEC 定义的执行，请编辑 snmptt.ini 文件并修改以下变量：

```bash
exec_enable
```

也可以在收到未知trap时启动外部程序。这可以通过在 **snmptt.ini** 中定义**unknown_trap_exec**来启用。传递给命令的将是所有标准的和企业的变量，类似于 **unknown_trap_log_file**但没有换行符。

# 运行模式

SNMPTT可以在两种模式下运行：独立模式和守护进程模式。

## 独立模式

要在独立模式下使用 SNMPTT，**snmptrapd.conf**文件将包含一个 **traphandle**，如下：

```bash
traphandle default /usr/sbin/snmptt
```

当 SNMPTRAPD 接收到trap时，trap将传递给 **/usr/sbin/snmptt** 脚本。SNMPTT 执行以下任务：

- 读取从 snmptrapd 传递的trap
- 加载包含trap定义的配置文件
- 搜索匹配的trap
- 记录、执行 EXEC 语句等
- 退出

使用 450 Mhz PIII 和包含 566 个唯一trap (EVENT) 的 9000 行 snmptt.conf，处理包括记录和执行 qpage 程序在内的trap需要不到一秒钟的时间。snmptt.conf 文件越大，处理时间越长。如果接收到大量trap，则应使用守护进程模式。如果处理一个trap需要 1 秒，那么显然您不应该尝试每秒处理超过一个trap。

运行不带 **--daemon** 命令行选项的 SNMPTT 将使用独立模式，除非在 **snmptt.ini** 文件中将 **mode** 设置为**daemon**。对于独立模式，**snmptt.ini** 文件中的 **mode** 变量应设置为 **standalone**。

注意：启用 UCD-SNMP / Net-SNMP Perl 模块会大大增加 SNMPTT 的启动时间。建议使用守护进程模式。

## 守护进程模式

当 SNMPTT 在守护进程模式下运行时，**snmptrapd.conf**文件将包含一个 **traphandle** 语句，如下：

```bash
traphandle default /usr/sbin/snmptthandler
```

当 SNMPTRAPD 接收到一个trap时，该trap将传递给 **/usr/sbin/snmpthandler** 脚本。SNMPTTHANDLER 执行以下任务：

- 读取从 snmptrapd 传递的trap
- 将trap写入到spool目录下的一个新的唯一文件中，例如 /var/spool/snmptt
- 退出

以守护进程模式运行的 SNMPTT 执行以下任务：

- 在启动时加载包含trap定义的配置文件
- 读取从spool目录传递来的trap
- 搜索匹配的trap
- 记录、执行 EXEC 语句等
- 休眠 5 秒（可配置）
- 循环回到“读取从spool目录传递来的trap”

在守护进程模式下使用 SNMPTTHANDLER 和 SNMPTT，每分钟可以轻松处理大量trap。

使用命令行选项 **--daemon** 运行 SNMPTT，或将 **snmptt.ini** 文件 中的 **mode** 变量设置为 **daemon** 可以使 SNMPTT 在守护进程模式下运行。

通过将**snmptt.ini** 的变量 **use_trap_time** 设置为**1**（默认），用于logging的日期和时间将是在trapspool文件中传递的日期和时间。如果**use_trap_time**设置为**0** ，则使用 SNMPTT处理trap的日期和时间。由于 SNMPTT 在spool目录轮询有休眠的时间长度，将**use_trap_time**设置为**0**可能会导致日志文件中的时间戳不准确。

注意：在非Windows 平台上运行时，如果 **daemon_fork**设置为 1 ，SNMPTT 将fork到后台并在 **/var/run/snmptt.pid**中创建一个 pid 文件。如果用户无法创建**/var/run /snmptt.pid**文件，它将尝试在当前工作目录中创建一个。

当作为守护进程运行时将 HUP 信号发送到 SNMPTT 将导致它重新加载配置文件，包括 .ini 文件、.ini 文件中列出的 snmptt.conf 文件以及任何 NODES 文件（如果 **dynamic_nodes**被禁用）。还可以通过将文件添加到名为**!reload**的spool目录来强制重新加载。文件名不区分大小写。如果检测到此文件，它将标记重新加载并删除该文件。这将是使用 Windows 时触发重新加载的唯一方法，因为 Windows 不支持信号。

可以通过将**statistics_interval snmptt.ini** 变量设置为大于 0 的值来启用 **total traps received**, **total traps translated**和 **total unknown traps** 的统计日志记录。在每个时间间隔（以秒为单位），统计信息将记录到 syslog 或 event log。

发送 USR1 信号还会记录 **total traps received**, **total traps translated** 和 **total unknown traps** 统计信息。例如，如果您想使用任务调度程序而不是使用**snmptt.ini**的变量 **statistics_interval** 中定义的间隔时间来记录统计信息，则可以使用此方法。还可以通过将文件添加到名为 **!statistics** 的spool目录来强制执行统计信息转储，该文件的处理类似于 **!reload** 文件。

## 命令行参数

支持以下命令行参数：

用法：
  snmptt [<options>]
选项：
  --daemon                         以守护进程模式启动
  --debug=n                        设置调试级别（1 或 2）
  --debugfile=filename     设置调试输出文件
  --dump                             转储（显示）定义的trap
  -- help                               显示此消息
  --ini=filename                  指定 snmptt.ini 文件的路径
  --version                           显示作者和版本信息
  --time                                用于查看加载和处理trap文件需要多长时间（例如：time snmptt --time）

# SNMPTT.CONF配置文件格式

配置文件（通常是 /etc/snmp/snmptt.conf 或 c:\snmp\snmptt.conf）包含所有已定义的traps的列表。

如果您的 snmptt.conf 文件变得相当大，并且您想将其分成许多较小的文件，请执行以下操作：

- 创建额外的 snmptt.conf 文件
- 将文件名添加到 snmptt.ini 文件的**snmptt_conf_files**部分。

例如：

```bash
snmptt_conf_files = <<END
/etc/snmp/snmptt.conf.generic
/etc/snmp/snmptt.conf.compaq
/etc/snmp/snmptt.conf.cisco
/etc/snmp/snmptt.conf.hp
/etc/snmp/snmptt.conf.3com
END
```

snmptt.conf 文件的语法是：

```bash
EVENT event_name event_OID "category" severity
FORMAT format_string

[EXEC command_string]

[NODES sources_list]

[MATCH [MODE=[or | and]] | [$n:[!][(    ) | n | n-n | > n | < n | x.x.x.x | x.x.x.x-x.x.x.x | x.x.x.x/x]]

[REGEX (    )(    )[i][g][e]]

[SDESC]
[EDESC]
```

注意：以 # 开头的行将被忽略。

注意： EVENT 和 FORMAT 行是必需的。[] 中的命令是可选的。不要在配置文件中包含 []！

## EVENT:

```bash
EVENT event_name event_OID "category" severity
```

**event_name:** 

​		不包含空格的唯一文本标签（别名）。当使用 **snmpttconvertmib** 进行转换时，这将匹配 MIB 文件中 TRAP-TYPE 或 NOTIFICATION-TYPE 行上的名称。

**event_OID:** 

​		以点分格式或不包含空格的符号表示法的对象标识符字符串。

​		例如，Compaq (enterprise .1.3.6.1.4.1.232) cpqHoGenericTrap trap (trap 11001) 可写为：.1.3.6.1.4.1.232.0.11001

​		如果通过在 snmptt.ini 文件中设置 **net_snmp_perl_enable** 安装并启用了 UCD-SNMP / Net-SNMP Perl 模块，也可以使用符号名称 。例如：

​				linkDown

​				IF-MIB::linkDown

​		注意：

​				Net-SNMP 5.0.9 及更早版本不支持在转换 OID 时包含模块名称（例如：**IF-MIB:: **）。Net-SNMP 5.1.1 及更高版本中包含的补丁适用于 5.0.8+。该补丁可从 [Net-SNMP 补丁页面](http://sourceforge.net/tracker/index.php?func=detail&aid=722075&group_id=12694&atid=312694) 获得。如果您使用的 Net-SNMP 版本不支持此功能，并且事件 OID 使用模块名称指定，则事件定义将被 **忽略**。另请注意，UCD-SNMP 可能无法将符号名称正确转换为数字 OID，这可能导致trap不匹配。

​				SNMP V1 trap的格式为企业 ID (.1.3.6.1.4.1.232) 后跟 0，然后是trap编号 (11001)。

​				配置文件中的同一个trap OID 可以有多个条目。如果在**snmptt.ini**中启用了**multiple_event**，那么它将处理所有匹配的trap。如果**multiple_event**被禁用，则仅使用第一个匹配条目。

​		也可以使用点分格式的通配符。例如： .1.3.6.1.4.1.232.1.2.*

​		注意：

​				特定trap匹配在通配符之前执行，因此如果您有 .1.3.6.1.4.1.232.1.2.5 和 .1.3.6.1.4.1.232.1.2.* 的条目，它会在收到时处理 .5 trap，即使首先定义通配符。只有在没有完全匹配的情况下才会匹配通配符匹配。这考虑了节点列表。因此，如果存在匹配trap，但 NODES 列表阻止将其视为匹配，则只有在没有其他完全匹配时才会使用通配符条目。

**category:**

​		用双引号括起来的字符串。在记录输出时使用（见上文）。

​		如果类别为“ **IGNORE** ”，即使 snmptt.conf 包含 FORMAT 或 EXEC 语句，也不会发生任何操作。

​		如果类别为“ **LOGONLY** ”，则trap将照常记录，但 EXEC 语句将被忽略。

​		注意：如果您计划使用 Nagios 等外部程序进行日志记录、分页等，您可能不希望使用 **LOGONLY** 定义任何trap，因为**EXEC**行永远不会用于提交被动服务检查。

**severity:**

​		事件严重性的字符串。在记录输出中使用。例如：Minor, Major, Normal, Critical, Warning。**snmptt.ini** 包含将系统日志级别或 NT 事件日志类型与严重级别匹配的选项。

## FORMAT:

```bash
FORMAT format_string
```

每个 EVENT 只能有一个 FORMAT 行。

format字符串用于生成将记录到任何受支持的记录方法的文本。

使用以下变量对此字符串执行变量替换：

$A     -  Trap agent主机名
$aA   -  Trap agent IP address
$Be   -  securityEngineID (snmpEngineID) 
$Bu   -  securityName (snmpCommunitySecurityName) 
$BE   -  contextEngineID (snmpCommunityContextEngineID) 
$Bn   -  contextName (snmpCommunityContextName) 
$c      -  类别
$C     -  Trap community 字段
$D    -  SNMPTT.CONF或MIB文件中的描述文本
$E     -  符号格式的企业trap OID
$e     -  数字格式的企业trap OID
$Fa   -  alarm (bell) (BEL)
$Ff    -  form feed (FF)
$Fn   -  newline (LF, NL)
$Fr    -  return (CR)
$Ft    -  tab (HT, TAB)
$Fz   -  转换的FORMAT行(EXEC only)
$G    -  通用trap编号（如果是企业trap，则为0）
$H    -  运行SNMPTT的系统的主机名
$S     -  特定trap编号（如果为通用trap，则为0）
$N    -  匹配的条目在.conf文件中定义的事件名称
$i      -  匹配的条目在.conf文件中定义的事件OID（可以是通配符OID）
$O    -  符号格式的trap OID
$o     -  数字格式的trap OID
$R, $r - Trap 主机名
$aR, $ar - IP地址
$s     -  严重性
$T     - Uptime: 网络实体初始化后的时间
$X     -  trap被后台处理的时间（守护程序模式）或当前时间（独立模式）
$x     -  trap被后台处理的日期（守护程序模式）或当前日期（独立模式）
$#    -  trap中的变量绑定数量（多少）
$$    -  输出 $
$@   - 自trap被后台处理（守护程序模式）或当前时间（独立模式）以来的秒数
$*n*    -  展开变量绑定 n (1-*n*) 
$+*n*  -  以 *variable name:value* 的格式展开变量绑定 n (1-*n*) 
$-*n*   -  以 *variable name (variable type):value* 的格式展开变量绑定 n (1-*n*) 
$v*n*  -  展开变量绑定 n (1-*n*) 的变量名称
$*    -  展开所有变量绑定
$+*  -  以 *variable name:value* 的格式展开所有变量绑定
$-*    - 以 *variable name (variable type):value* 的格式展开所有变量绑定

例子：

```bash
FORMAT NIC switchover to slot $3, port $4 from slot $5, port $6
```

注意：

对于文本日志文件，输出将被格式化为：

```bash
date time trap-OID severity category hostname - format
```

对于除 MySQL、DBD::ODBC 和 Win32::ODBC 之外的所有其他日志文件，输出将被格式化为：

```bahs
trap-OID severity category hostname - format
```

对于 MySQL、DBD::ODBC 和 Win32::ODBC，**formatline**列将仅包含**format**文本。

注（1）：

如果 在 **snmptt.ini** 文件中启用 **translate_integers**，则 SNMPTT 将尝试通过在 MIB 文件中执行查找将trap中接收的整数值转换为文本。 您必须安装 UCD-SNMP / Net-SNMP Perl 模块才能使其工作，并且必须通过 在 snmptt.ini 文件中启用**net_snmp_perl_enable来启用对它的支持。**

要使此功能正常工作，您必须确保使用所有必需的 MIBS 正确配置 UCD-SNMP / Net-SNMP。如果启用该选项，但找不到该值，则将使用整数值。如果 MIB 文件存在，但未发生转换，请确保正确配置 UCD-SNMP / Net-SNMP 以处理所有必需的 mib。这是在 UCD-SNMP / Net-SNMP **snmp.conf**文件中配置的。或者，您可以尝试将 snmptt.ini 中的**mibs_enviroment** 变量**设置**为**ALL**（无引号），以强制在 SNMPTT 启动时初始化所有 MIBS。

如果在使用独立模式时启用**translate_integers** ，则由于 MIB 文件的初始化，处理每个trap可能需要更长的时间。

注（2）：

$v*n*, $+*n* 和 $-*n* 变量名称和变量类型通过在 MIB 文件中执行查找转换为文本名称。您必须安装 UCD-SNMP / Net-SNMP Perl 模块才能使其工作，并且必须通过在 snmptt.ini 文件中启用 net_snmp_perl_enable来启用对它的支持。

如果 net_snmp_perl_enable 未启用，则 $vn 变量将替换为文本“variable*n*”，其中n是变量编号 (1+)。

要使名称转换起作用，您必须确保使用所有必需的 MIBS 正确配置 UCD-SNMP / Net-SNMP。如果启用该选项并且未返回正确的名称，请确保正确配置 UCD-SNMP / Net-SNMP 以处理所有必需的 mib。这是在 UCD-SNMP / Net-SNMP 的 **snmp.conf**文件中配置的。或者，您可以尝试将 **snmptt.ini** 中的**mibs_enviroment** 变量设置为 **ALL**（无引号），以强制在 SNMPTT 启动时初始化所有 MIBS。

注（3）：

如果在**snmptt.ini** 文件中启用 **translate_trap_oid**，则 SNMPTT 将尝试将接收到的trap的数字 OID 转换为符号形式，例如 IF-MIB::linkDown。您必须安装 UCD-SNMP / Net-SNMP Perl 模块才能使其工作，并且必须通过在 snmptt.ini 文件中启用**net_snmp_perl_enable** 来启用对它的支持。如果**net_snmp_perl_enable** 未启用，它将默认使用数字 OID。 

如果 在 **snmptt.ini** 文件中启用**translate_oids**，则 SNMPTT 将尝试将在trap内传递的变量中找到的任何数字 OID 转换为符号形式。您必须安装 UCD-SNMP / Net-SNMP Perl 模块才能使其工作，并且必须通过在 snmptt.ini 文件中启用**net_snmp_perl_enable** 来启用对它的支持。如果**net_snmp_perl_enable**未启用，它将默认使用数字 OID。

Net-SNMP 5.0.9 及更早版本不支持包含模块名称（例如：**IF-MIB::**) 在翻译 OID 时，大多数 5.0.x 版本不能正确地将数字 OID 翻译成长符号名称。Net-SNMP 5.1.1 及更高版本中包含的补丁适用于 5.0.8+。该补丁可从[ Net-SNMP 补丁页面](http://sourceforge.net/tracker/index.php?func=detail&aid=722075&group_id=12694&atid=312694)获得。

注（4）：

snmptt.ini **description_mode**选项必须设置为 1 或 2。如果设置为 1，则从 SNMPTT.CONF 文件中提取描述**。** 如果设置为 2，则从 MIB 文件中提取描述。如果使用 MIB 文件，您必须安装并启用 UCD-SNMP / Net-SNMP Perl 模块。

注（7）：

这些变量仅在使用 snmptrapd 的嵌入式trap处理程序（snmptthandler-embedded）时可用。

## EXEC:

```bash
[EXEC command_string]
```

每个 EVENT 可以有多个 EXEC 行。

可选字符串，包含接收到trap时执行的命令和传递给程序的参数。EXEC 行按照它们出现的顺序执行。

EXEC 使用与 FORMAT 行相同的变量替换。

例子：

```bash
EXEC /usr/bin/qpage -f TRAP alex "$r: $x $X - NIC switchover to slot $3, port $4 from slot $5, port $6"
```

或者

```bash
EXEC c:\snmp\pager netops "$r: $x $X - NIC switchover to slot $3, port $4 from slot $5, port $6"
```

注意：与 FORMAT 行不同，消息前面没有任何内容。如果您想在上面的页面中包含主机名和日期，则必须使用 $r、$x 和 $X 等变量。

注意：如果在 snmptt.conf 文件中将trap严重性设置为 LOGONLY，则不会执行 EXEC。

## PREEXEC:

```bash
[PREEXEC command_string]
```

每个 EVENT 可以有多个 PREEXEC 行。

可选字符串，包含在接收到trap之后但在处理 FORMAT 和 EXEC 语句之**前**要执行的命令。外部程序的输出存储在 **$p*n*** 变量中，其中 ***n*** 是从 1 开始的数字。允许多个 PREEXEC 行。第一个 PREEXEC 将命令的结果存储在 **$p1** 中，第二个在 **$p2** 中等等。所有结束的换行符都被删除。snmptt.ini 的参数 **pre_exec_enable** 可用于启用或禁用**PREEXEC** 语句。

**PREEXEC**使用与 FORMAT 行相同的变量替换。

示例：

```bash
EVENT linkDown .1.3.6.1.6.3.1.1.5.3 "Status Events" Normal
FORMAT Link down on interface $1($p1). Admin state: $2. Operational state: $3
PREEXEC /usr/local/bin/snmpget -v 1 -Ovq -c public $aA ifDescr.$1
```

示例输出：

```bash
Link down on interface 69("100BaseTX Port 1/6 Name SERVER1"). Admin state up. Operational state: down
```

在上面的示例中，结果用引号引起来，因为这是从 snmpget 返回的（它不是由 SNMPTT 添加的）。

注意：即使在 snmptt.conf 文件中将trap严重性设置为 LOGONLY，PREEXEC 也会执行。

NODES:

```bash
[NODES sources_list]
```

用于限制哪些设备可以映射到此 EVENT 定义。 

每个 EVENT 可以有多个 NODES 行。

可选字符串，包含主机名、IP 地址、CIDR 网络地址、网络 IP 地址范围或文件名的任意组合。如果省略此关键字，则将接受所有来源。检查每个条目是否匹配。一旦发生匹配，搜索就会停止。

例如，如果您只希望子网 192.168.1.0/24 上的设备触发此事件，您可以使用 NODES 条目：

```bash
NODES 192.168.1.0/24
```

如果指定了文件名，则必须使用完整路径指定。 

有两种操作模式：**POS**（positive - 默认）和**NEG**（negative）。如果设置为**POS**，则如果有任何 **NODES** 条目匹配，则**NODES**是“匹配” 。如果设置为**NEG**，则仅当没有任何**NODES**条目匹配时，**NODES**才是“匹配” 。要更改操作模式，请使用以下语句之一：

```bash
NODES MODE=POS
NODES MODE=NEG
```

此功能的常见用途是当您的设备以非标准方式（例如添加其他变量）实现trap时，例如 linkDown 和 linkUp trap。通过定义两个 EVENT 语句并将 NODES 语句与 NODES MODE 一起使用，您可以让一个 EVENT 语句处理标准设备，而另一个使用扩展的 linkDown / linkUp trap处理其他设备。

示例 1：此示例将匹配任何名为**fred**、 **barney**、**betty**或**wilma**的主机：

```bash
NODES fred barney betty wilma
```

示例 2：此示例将匹配任何 **不** 为 **fred**、**barney**、**betty**或**wilma**的主机：

```bash
NODES fred barney betty wilma
NODES MODE=NEG
```

示例 3：

此示例将加载文件 /etc/snmptt-nodes（见下文），并匹配任何名为**fred**、**barney**、**betty**的主机，网络 IP 地址 **192.168.1.1、192.168.1.2、192.168.1.3、192.168.2.1**、网络范围**192.168 .50.0/22**或网络范围 **192.168.60.0-192.168.61.255** :

```bash
NODES /etc/snmptt-nodes
```

示例 4：

此示例将加载 **/etc/snmptt-nodes**和**/etc/snmptt-nodes2** 两个文件（参见上面的示例）：

```bash
NODES /etc/snmptt-nodes /etc/snmptt-nodes2
```

示例 5：

```bash
NODES 192.168.4.0/22 192.168.60.0-192.168.61.255 /etc/snmptt-nodes2
```

示例 6：

```bash
NODES fred /etc/snmptt-nodes pebbles /etc/snmptt-nodes2 barney
```



其中 snmptt-nodes 包含：

```bash
fred
barney betty
# comment lines
192.168.1.1 192.168.1.2 192.168.1.3
192.168.2.1
192.168.50.0/22
192.168.60.0-192.168.61.255
wilma
```

注意：

- 名称不区分大小写，注释行允许以 # 开头。
- CIDR 网络地址必须使用 4 个八位字节后跟 / 再后跟位数来指定。例如：172.16.0.0/24。使用 172.16/24 将不起作用。
- 不要在网络范围之间使用任何空格，因为它们将被解释为两个不同的值。例如，192.168.1.1 - 192.168.1.20 将不起作用。请改用 192.168.1.1-192.168.1.20。
- 默认情况下，在加载 snmptt.conf 文件时（在 SNMPTT 启动期间）加载 NODES 文件。snmptt.ini选项的**dynamic_nodes** 可以设置为  **1** 以在每次处理 EVENT 时加载节点文件。
- 有关重要的 DNS 信息，请参阅“[名称解析/DNS ](http://snmptt.sourceforge.net/docs/snmptt.shtml#DNS)”部分。

## MATCH:

```bash
[MATCH [MODE=[or | and]] | [$n:[!][(    )[i] | n | n-n | > n | < n | x.x.x.x | x.x.x.x-x.x.x.x | x.x.x.x/x]]
```

可选的匹配表达式，必须将其计算为 true 才能将trap视为与此 EVENT 定义的匹配。

如果 MATCH 语句存在，并且没有匹配项评估为 true，则默认值将不匹配此 EVENT 定义。

支持以下 Perl 正则表达式修饰符：

​		**i** - 尝试匹配时忽略大小写

可以使用以下命令格式：

```bash
MATCH MODE=[or | and]
MATCH $x: [!] (reg) [i]
MATCH $x: [!] n
MATCH $x: [!] n-n
MATCH $x: [!] < n
MATCH $x: [!] > n
MATCH $x: [!] & n
MATCH $x: [!] x.x.x.x
MATCH $x: [!] x.x.x.x-x.x.x.x
MATCH $x: [!] x.x.x.x/x
```

其中：

​	**or** 或 **and** 为所有匹配设置默认评估模式
​	**$x** 是任何变量（例如：$3、$A 等）
​	**reg** 是正则表达式
​	**！**用于否定结果（非）
​	**&** 用于执行按位与
​	**n** 是数字
​	**x.x.x.x** 是 IP 地址
​	**x.x.x.x-x.x.x.x** 是 IP 地址范围
​	**x.x.x.x/x**是 IP CIDR 地址

注意：

- 要根据发送trap的设备/代理的 IP 地址/主机名限制哪些设备可以映射到此 EVENT 定义，建议使用**NODES**关键字。
- 如果匹配模式为“or”，则一旦发生匹配，则不执行其他匹配，最终结果为真。
- 如果匹配模式为'and'，一旦匹配失败，则不执行其他匹配，最终结果为假。
- 要在搜索表达式中使用括号 ( 或 ) ，它们必须使用反斜杠转义 (\\)。
- 如果不存在 MATCH MODE= 行，则默认为“or”。
- 每个 EVENT 只能有一种匹配**模式。**如果存在多个 MATCH MODE= 行，则使用列表中的最后一个。

例子：

​	$2 必须介于 1000 和 2000 之间：

```bash
MATCH $2: 1000-2000
```

任何一个匹配即可（or）：$3 必须是 52，或者 $4 必须是 192.168.1.10 和 192.168.1.20 之间的 IP 地址，或者严重性必须是“主要”：

```bash
MATCH $3: 52
MATCH $4: 192.168.1.10-192.168.1.20
MATCH $s: (Major)
```

全部必须匹配（and）：$3 必须大于 20，$5 不能包含单词 alarm 或 critical，$6 必须包含字符串 '(1) remaining'，$7 必须包含不区分大小写的字符串 'power' ：

```bash
MATCH $3: >20
MATCH $5: !(alarm|critical)
MATCH $6: (\(1\) remaining)
MATCH $7: (power)i
MATCH MODE=and
```



整数 $1 必须设置第 4 位：

```bash
MATCH $1: &8
```

## REGEX:

```bash
[REGEX(    )(    )[i][g][e]]
```

可选正则表达式，用于在**转换后**的FORMAT / EXEC 行上执行搜索和替换。允许多个 REGEX ( )( ) 行。

第一个 ( ) 包含搜索表达式。
第二个 ( ) 包含替换文本

支持以下 Perl 正则表达式修饰符：

​	**i** - 尝试匹配左侧时忽略大小写
​	**g** - 替换所有出现而不是仅替换第一个
​	**e** - 将右侧（eval）作为代码执行

要使用捕获（memory parenthesis）或**e**修饰符进行替换，您必须先在 snmptt.ini 文件中将**allow_unsafe_regex**设置为**1**启用支持。注意： **这被认为是不安全的，因为正确的表达式的内容是由 Perl 执行 (eval) 的，其中可能包含不安全的代码**。如果启用此选项， **请确保 SNMPTT 配置文件是安全的！** 

每个 REGEX 行从上到下按顺序处理，并且是累积的。第二个 REGEX 对第一个 REGEX 的结果进行操作。

例子：

```bash
FORMAT line before:  UPS has       detected a      building alarm.       Cause: UPS1 Alarm #14: Building alarm 3.

REGEX (Building alarm 3)(Computer room high temperature)
REGEX (Building alarm 4)(Moisture detection alarm)
REGEX (roOm)(ROOM)ig
REGEX (UPS)(The big UPS)
REGEX (\s+)( )g

FORMAT line after:  The big UPS has detected a building alarm. Cause: UPS1 Alarm #14: Computer ROOM high temperature
```

要在搜索表达式中使用括号 ( or )，它们必须加反斜杠 (\)，否则将被解释为捕获（见下文）。替换文本不需要反斜杠。

例子：

```bash
FORMAT line before:  Alarm (1) and (2) has been triggered

REGEX (\(1\))(One)
REGEX (\(2\))((Two))

FORMAT line after:  Alarm One and (Two) has been triggered
```

如果启用了**allow_unsafe_regex**，则可以在替换文本中使用捕获。

例子：

```bash
FORMAT line before:  The system has logged exception error 55 for the service testservice

REGEX (The system has logged exception error (\d+) for the service (\w+))(Service $2 generated error $1)

FORMAT line after:  Service testservice generated error 55
```


如果启用了**allow_unsafe_regex** 并指定了 **e** 修饰符，则执行（评估）右侧。这允许您使用 Perl 函数执行各种任务，例如从十六进制转换为十进制，使用 sprintf 格式化文本等。所有文本必须在引号内，并且可以使用点 (.) 将语句连接在一起。

示例 1：

```bash
FORMAT line before:  Authentication Failure Trap from IP address: C0 A8 1 FE

REGEX (Address: (\w+)\s+(\w+)\s+(\w+)\s+(\w+))("address: ".hex($1).".".hex($2).".".hex($3).".".hex($4))ei

FORMAT line after:  Authentication Failure Trap from IP address: 192.168.1.254
```

示例 2：

```bash
FORMAT line before:  Authentication Failure Trap from IP address: C0 A8 1 FE

REGEX (Address: (\w+)\s+(\w+)\s+(\w+)\s+(\w+))("address:".sprintf("%03d.%03d.%03d.%03d",hex($1),hex($2),hex($3),hex($4)))ie

FORMAT line after:  Authentication Failure Trap from IP address: 192.168.001.254
```

示例 3

此示例适用于 BGP **bgpBackwardTranstion** trap。bgpBackwardTranstion trap的 OID 将转换后的设备的 IP 地址附加到 OID 的末尾。要创建有意义的trap消息，IP 地址需要与变量 OID 分开。因为 IP 地址是 OID **变量名称**而不是 OID **值**的一部分，所以需要一个 REGEX 表达式。下面使用 FORMAT 行上的 $+1 变量，以便 REGEX 可以解析出 IP 地址。 

```bash
FORMAT line before:  Peer:$+2

FORMAT line after substitution, but before REGEX:  Peer:bgpPeerState.192.168.1.1:idle

REGEX (Peer:.*\.(\d+\.\d+\.\d+\.\d+):(.*))("Peer: $1 has transitioned to $2")e

FORMAT line after:  Peer: 192.168.1.1 has transitioned to idle
```

示例 4

此示例是在**REGEX**语句中使用 Perl 子例程的示例。

```bash
FORMAT line before:  Extremely severe error has occured

REGEX (Extremely severe error has occured)(("Better get a lotto ticket!!  Here is a lotto number to try:".sprintf ("%s", lottonumber());sub lottonumber { for(my $i=0;$i<6;$i++) { $temp = $temp . " " . (int(rand 49) +1); } return $temp; } )ie

FORMAT line after:  Better get a lotto ticket!!  Here is a lotto number to try: 36 27 38 32 29 6
```

注意：在完成所有变量替换后，REGEX 表达式在最终转换的 FORMAT / EXEC 行上执行。

## SDESC

```bash
[SDESC]
```

描述的可选开头。snmptt 将忽略此行和 EDESC 行之间的所有文本。此部分可用于输入有关trap的注释以供您自己使用。如果你使用 SDESC，你必须跟着一个 EDESC。

## EDESC

```bash
[EDESC]
```

用于结束描述部分。

例子：

```bash
SDESC
Trap used when power supply fails in
a server.
EDESC
```

# SNMPTT.CONF 配置文件注意事项

当配置文件中有同一个trap的多个定义时，适用以下规则：

**匹配发生在：**

- 接收到的trap OID 与配置文件中定义的 OID 匹配
- **AND** **（**主机名与 NODES 条目中定义的主机名匹配，**OR** 没有 NODES 条目**）**
- **AND** **（** MATCH 语句的计算结果为 TRUE **OR** 没有 MATCH 条目 **）**

**如果在 snmptt.ini 中将 multiple_event 设置为 1：**

- trap的处理次数与配置文件中匹配的次数一样多
- 如果存在任意数量的完全匹配，则不执行通配符匹配
- 如果不存在完全匹配，则在以下情况下执行通配符匹配（主机名与 NODES 条目中定义的主机名匹配，**OR** 没有 NODES 条目） **并且** （ MATCH 语句的计算结果为 TRUE，**OR** 没有 MATCH 条目 ）

**如果在 snmptt.ini 中将 multiple_event 设置为 0：**

- 使用配置文件中的第一个匹配项处理一次trap
- 如果存在完全匹配，则不执行通配符匹配
- 如果不存在完全匹配，则在以下情况下执行通配符匹配（主机名与 NODES 条目中定义的主机名匹配，**OR** 没有 NODES 条目） **并且** （ MATCH 语句的计算结果为 TRUE，**OR** 没有 MATCH 条目 ）

# 名称解析/DNS

Snmptrapd 传递发送trap的设备的 IP 地址（主机）、发送trap的设备的主机名（主机）（如果配置为解析主机名）和实际 SNMP agnet（agent）的 IP 地址。

如果配置将 **dns_enable**设置为 0（禁用 dns），则 AGENT 的主机名将不可用于 $A 变量、NODES 匹配以及 SQL 数据库中的主机名列。唯一的例外是（主机）IP 地址与（代理）IP 地址匹配，并且 snmptrapd 被配置为解析主机名。在这种情况下，（主机）的主机名将用于（代理）主机名，因为它们显然是相同的主机。

如果配置将 **dns_enable**设置为 1（启用 dns），则主机和代理的主机名将通过 DNS 解析。在执行匹配之前，NODES 条目也将解析为 IP 地址。

主机名可以解析为完全限定域名 (FQDN)。例如：barney.bedrock.com。在 /etc/hosts 文件或 %systemroot%\system32\drivers\etc\hosts 中添加主机条目可能会导致使用短名称 (barney)。您还可以启用 **strip_domain** / **strip_domain_list**选项让 SNMPTT 剥离任何 FQDN 主机的域。有关详细信息，请参阅 **snmptt.ini**文件。

要允许将 IP 地址解析为主机名，DNS 中必须存在 PTR 记录，或者本地主机文件必须包含所有主机。

**建议在运行 SNMPTT/snmptrapd 的机器上安装 DNS，或者为所有设备配置本地主机文件**。DNS 应配置为接收trap的域的从属（权威）。这将减少网络解析流量，加快解析速度，并消除网络对 DNS 的依赖。 **如果不使用本地 DNS 或 hosts 文件，那么在 DNS / 远程网络中断期间，整个网络管理站可能会变得无用，并可能导致网络管理软件误报。**

# SNMPTT.CONF 文件示例1

注意：**示例** 文件夹还包含一个示例**snmptt.conf**文件。

以下是 **snmptt.conf** 中定义的两个trap的示例：

```bash
#
EVENT COMPAQ_11003 .1.3.6.1.4.1.232.0.11003 "LOGONLY" Normal
FORMAT Compaq Generic Trap: $*
EXEC qpage -f TRAP notifygroup1 "Compaq Generic Trap: $*"
NODES /etc/snmp/cpqnodes
SDESC
Generic test trap
EDESC
#
#
EVENT cpqDa3AccelBatteryFailed .1.3.6.1.4.1.232.0.3014 "Error Events" Critical
FORMAT Battery status is $3.
EXEC qpage -f TRAP notifygroup1 "$s $r $x $X: Battery status is $3"
NODES ntserver1 ntserver2 ntserver3
#
#
```

# SNMPTT.CONF 文件示例2

以下是要在 **snmptt.ini** 中加载文件列表的示例：

```bash
snmptt_conf_files = <<END
/etc/snmp/snmp-compaq.conf
/etc/snmp/snmp-compaq-hsv.conf
END
```

以下是 **/etc/snmp/snmptt-compaq.conf** 中定义的一个trap的示例：

```bash
#
EVENT COMPAQ_11003 .1.3.6.1.4.1.232.0.11003 "LOGONLY" Normal
FORMAT Compaq Generic Trap: $*
EXEC qpage -f TRAP notifygroup1 "Compaq Generic Trap: $*"
NODES /etc/snmp/cpqnodes
SDESC
Generic test trap
EDESC
#
```

以下是 **/etc/snmp/snmptt-compaq-hsv.conf** 中定义的一个trap的示例：

```bash
#
EVENT mngmtAgentTrap-16025 .1.3.6.1.4.1.232.0.136016025 "Status Events" Normal
FORMAT Host $1 : SCellName-TimeDate $2 : EventCode $3 : Description $4
EXEC qpage -f TRAP notifygroup1 "Host $1 : SCellName-TimeDate $2 : EventCode $3 : Description $4"
SDESC
"Ema EMU Internal State Machine Error [status:10]"
EDESC
#
```

# 注意

大多数情况下可以使用现有的 HP Openview trapd.conf，但该文件必须是 VERSION 3 文件。SNMPTT 不支持 HPOV 中实现的所有变量，但大多数都可用。以下变量可能与 HPOV 完全匹配，也可能不完全匹配：$O、$o、$r、$ar、$R、$aR。

一些供应商（例如 Compaq 和 Cisco ）提供了一个文件，可以使用 HP Openview 实用程序将其导入 HP Openview。 **Snmpttconvert**可用于将文件转换为**snmptt.conf**格式。

一些供应商提供包含 TRAP 或 NOTIFICATION 定义的 MIB 文件。 **snmpttconvertmib**可用于将文件转换为**snmptt.conf** 格式。


# 限制

### 仅独立模式：

使用 450 Mhz PIII 和包含 566 个唯一trap (EVENT) 的 9000 行的 snmptt.conf，处理包括记录和执行 qpage 程序在内的trap需要不到一秒钟的时间。snmptt.conf 文件越大，处理时间越长。如果接收到大量trap，则应使用守护进程模式。如果处理一个trap需要 1 秒，那么显然您不应该尝试每秒处理超过一个trap。

注意：启用 UCD-SNMP / Net-SNMP Perl 模块会大大增加 SNMPTT 的启动时间。建议使用守护进程模式。

### **独立或守护进程模式：**

执行 traphandle 命令时 snmptrapd 程序会阻塞。这意味着如果调用的程序永不退出，SNMPTRAPD 将永远等待。如果在 traphandler 运行时接收到一个trap，它会被缓冲并在 traphandler 完成时进行处理。但是我不知道这个缓冲区有多大。

由 SNMPTT (EXEC) 调用的程序会阻止 SNMPTT。如果您调用一个不返回的程序，SNMPTT 将等待。在独立模式下，这也会导致 snmptrapd 永远等待。
