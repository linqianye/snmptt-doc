# SNMPTTCONVERT

一些供应商提供的文件可以使用HP Openview实用程序导入HP Openview。。 **SNMPTTCONVERT** 是一个简单的 Perl 脚本，它将其中一个文件转换为 SNMPTT 使用的格式。该文件可以包含多个陷阱。

例如，如果文件 ciscotrap.txt 包含：

```bash
rpsFailed {.1.3.6.1.4.1.437.1.1.3} 6 5 - "Status Events" 1
Trap received from enterprise $E with $# arguments: sysName=$1
SDESC
"A redundant power source is connected to the switch but a failure exists in
the power system."
EDESC
```

执行 snmpttconvert ciscotrap.txt 将输出：

```bash
#
#
#
EVENT rpsFailed .1.3.6.1.4.1.437.1.1.3.0.5 "Status Events" Normal
FORMAT Trap received from enterprise $E with $# arguments: sysName=$1
#EXEC qpage -f TRAP notifygroup1 "Trap received from enterprise $E with $# arguments: sysName=$1"
SDESC
"A redundant power source is connected to the switch but a failure exists in
the power system."
EDESC
```

注意：默认情况下会添加#EXEC 行。这可以通过编辑 SNMPTTCONVERT 脚本来更改。

