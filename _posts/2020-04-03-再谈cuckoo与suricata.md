---
title: 再谈cuckoo与suricata
tags: suricata
categories: 沙箱
---


* TOC
{:toc}

---------------

## 再谈cuckoo与suricata

suricata 与cuckoo可以有机的进行结合。suricata还原的文件可以给cuckoo使用，cuckoo中恶意文件产生的数据流量报文，可以反过来给suricata回放规则检测。
二者可以有机进行结合。 突然想到opensense与suricata也可以有机结合呢。

## suricata到底可以做哪些工作？


