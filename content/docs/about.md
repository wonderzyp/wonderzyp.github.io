---
bookHidden: true
title: "About Me"
bookToc: false
---

## 简述
23年硕士毕业，工作聚焦于linux内核，主涉及内核稳定性的问题分析与工具开发。

## 工作经历

- **上海蔚来汽车有限公司，数字座舱与软件开发，固件软件研发工程师，2023.7~今**

## 项目经历

- **NT3.0平台稳定性工具开发**

- 针对解析dump环境搭建复杂问题，基于docker+jenkins搭建dump可视化解析平台，支持QNX/Android的Ramdump, Coredump与minidump解析。有效提升内部研发团队分析问题效率。
- 开发crash-utility插件[crash-qcom-ipclog](https://github.com/wonderzyp/crash-qcom-ipclog)，支持解析出高通ipclog。
- 开发[TZ log解析工具](https://github.com/wonderzyp/dump_tzlog)，支持Ramdump/Minidump的tz log解析。
- 开发QNX Ramdump提取Android Guestdump工具：[ramdump2gcore](https://github.com/wonderzyp/ramdump2gcore)
- 针对crash-utility在KASAN版本下崩溃问题，提出patch并合入社区主线。

- **NT3.0平台稳定性问题分析**

    分析调试内核相关问题，相应技术文档总结可见[此博客](https://wonderzyp.github.io/)

- **CFD软件并行化开发与移植**

    针对传统串行CFD程序计算量大，计算耗时长的问题，将其并行化改造，并部署至曙光超算平台。
    - 相较于原始串行程序，并行化的程序在512核心下获得约568倍加速比
    - 主要涉及：C++,MPI,多进程的gdb调试。

## 教育经历
- **2020年9月** 至 **2023年7月**：大连理工大学 车辆工程
- **2016年9月** 至 **2020年6月**：合肥工业大学 智能车辆实验班