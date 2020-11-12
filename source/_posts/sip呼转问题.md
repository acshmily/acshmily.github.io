---
title: sip呼转问题
date: 2020-11-05 16:38:07
tags:
	-	SIP
categories:	工作笔记
---

# 关于SIP语音呼转问题

​	记录一下，模拟大网呼转的时候History-info 字段必须规范，必须使用sip tag 包装主叫、被叫、并且携带IMS@域。不可用tel tag.

