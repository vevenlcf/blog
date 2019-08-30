---
title: Procmon.exe在cuckoo中的使用
tags: Procmon,cuckoo 
categories: Cuckoo
---

## 背景
最近研究了下procmon.exe，该工具用途可大了，Procmon是微软出品用于监视Windows系统里程序的运行情况，监视内容包括该程序对注册表的读写、
对文件的读写、网络的连接、进程和线程的调用情况，procmon 是一款超强的系统监视软件。

## Cuckoo中运用
cuckoo中使用该工具在agent客户机上进行辅助监测功能，用以获取注册表、文件读写、网络连接等信息。
其运行过程为：
   ```开启procmon.exe  --> 开启执行程序 --> 停止procmon.exe ---> 加载配置文件同时保存信息到xml ---> 对xml格式信息进行解析```

## 运行指令
  1. ```Procmon.exe /AcceptEula /Minimized /BackingFile procmon.pml```
  2. ```Procmon.exe /Terminate```
  3. ```Procmon.exe /OpenLog procmon.pml /LoadConfig procmon.pmc /SaveAs procmon.xml /SaveApplyFilter```

  ![QQ截图20190830095743.jpg](https://i.loli.net/2019/08/30/O1ShozfLc9IwQ7E.jpg)

## cuckoo源代码
### cuckoo agent客户端代码
```python
import os.path
import subprocess
import time

from lib.common.abstracts import Auxiliary
from lib.common.exceptions import CuckooDisableModule, CuckooPackageError
from lib.common.results import upload_to_host

class Procmon(Auxiliary):
    """Allow procmon to be run on the side."""
    def start(self):
        if not self.options.get("procmon"):
            raise CuckooDisableModule

        bin_path = os.path.join(self.analyzer.path, "bin")

        self.procmon_exe = os.path.join(bin_path, "procmon.exe")
        self.procmon_pmc = os.path.join(bin_path, "procmon.pmc")
        self.procmon_pml = os.path.join(bin_path, "procmon.pml")
        self.procmon_xml = os.path.join(bin_path, "procmon.xml")

        if not os.path.exists(self.procmon_exe) or \
                not os.path.exists(self.procmon_pmc):
            raise CuckooPackageError(
                "In order to use the Process Monitor functionality it is "
                "required to have Procmon setup with Cuckoo. Please run the "
                "Cuckoo Community script which will automatically fetch all "
                "related files to get you up-and-running."
            )

        # Start process monitor in the background.
        subprocess.Popen([
            self.procmon_exe,
            "/AcceptEula",
            "/Quiet",
            "/Minimized",
            "/BackingFile", self.procmon_pml,
        ])

        # Try to avoid race conditions by waiting until at least something
        # has been written to the log file.
        while not os.path.exists(self.procmon_pml) or \
                not os.path.getsize(self.procmon_pml):
            time.sleep(0.1)

    def stop(self):
        # Terminate process monitor.
        subprocess.check_call([
            self.procmon_exe,
            "/Terminate",
        ])

        # Convert the process monitor log into a readable XML file.
        subprocess.check_call([
            self.procmon_exe,
            "/OpenLog", self.procmon_pml,
            "/LoadConfig", self.procmon_pmc,
            "/SaveAs", self.procmon_xml,
            "/SaveApplyFilter",
        ])

        # Upload the XML file to the host.
        upload_to_host(self.procmon_xml, os.path.join("logs", "procmon.xml"))
```

### cuckoo server端部分代码
```python
import os.path
import xml.etree.ElementTree

from cuckoo.common.abstracts import Processing

class ProcmonLog(list):
    """Yields each API call event to the parent handler."""

    def __init__(self, filepath):
        list.__init__(self)
        self.filepath = filepath

    def __iter__(self):
        iterator = xml.etree.ElementTree.iterparse(
            open(self.filepath, "rb"), events=["end"]
        )
        for _, element in iterator:
            if element.tag != "event":
                continue

            entry = {}
            for child in element.getchildren():
                entry[child.tag] = child.text
            yield entry

    def __nonzero__(self):
        # For documentation on this please refer to MonitorProcessLog.
        return True

class Procmon(Processing):
    """Extracts events from procmon.exe output."""

    key = "procmon"

    def run(self):
        procmon_xml = os.path.join(self.logs_path, "procmon.xml")
        if not os.path.exists(procmon_xml):
            return

        return ProcmonLog(procmon_xml)
```
## 裁剪的代码
```python
"""
@author: vevenlcf
@time: 20190830
"""
import os.path
import xml.etree.ElementTree


def ProcMonParse():
    iterator = xml.etree.ElementTree.iterparse(
        open("procmon.xml", "rb"), events=["end"]
    )
    entry = {}
    for _, element in iterator:
        if element.tag != "event":
            continue

        for child in element.getchildren():
            entry[child.tag] = child.text
            print(entry)
    #   yield entry

a=ProcMonParse()

```
