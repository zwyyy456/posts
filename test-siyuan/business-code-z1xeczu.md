---
title: 业务代码
slug: business-code-z1xeczu
url: /post/business-code-z1xeczu.html
date: '2024-09-03 09:31:27+08:00'
lastmod: '2024-09-05 17:49:28+08:00'
toc: true
isCJKLanguage: true
---

# 业务代码

## 简介

主要就是记录一下风控终端中的那些按钮，在按下之后发出的消息包的 TID，以及对应的逻辑处理函数。

关于不同的 TID 的 package 对应的消息内容，可以在 `XTPMisc.xml` 中查看，位于 `datamodel/env` 目录下。

## 强平报单

对应的 TID 为 `ReqRiskOrderInsert`，对应的 case 为 `RMS_TID_ReqRiskOrderInsert`，对应的函数实现为 `CRiksServiceImpl::ReqRiskOrderInsert`。

`ReqRiskOrderInsert` 的执行次数等于点击 `强平报单` 时勾选的强平单数量。

### 预埋强平单处理流程

强平报单后，会发出 `RMS_TID_ReqOrderInsert` 或者 `RMS_TID_ReqParkedOrderInsert`，这里主要讨论预埋单的问题，即 `RMS_TID_ReqParkedOrderInsert`，`RiskApiMgr` 收到该 package 后，调用 `ReqParkedOrderInsert(inputOrder)`，该函数会调用交易的 API，将单子报给交易。

交易处理之后（对于预埋单，暂存到一个表中，交易所开启后才会报给交易所），返回一个 `FTD_TID_RspParkedOrderAction` 给 `RiskApiMgr`，使其调用 `CRiskApiCallback::OnRspParkedOrderInsert`，从而向流中发送 `RMS_TID_RspParkedOrderInsert` package，`TradeTriggered` 收到这个 package 之后，就会调用 `UpdateForceClosingInvByParkedRsp` ，真正的实现位于 `AutoRemindHandler.cpp` 中，去将该 `InvestorID` 插入到 `m_ForceClosingInvSet`，

### 强平中的出入金提醒

对强平单报单到成交这个过程中，如果发生了出入金，就是检查发生了出入金的账户是否存在于 `m_ForceClosingInvSet` 中。出入金的处理在 `TradeTriggered` 的 `case FID_UstpInvestorAccountDeposit`中，会调用 `CheckForceClosingTodayInout` 来检查强平报单后是否发生出入金，如果发生了，会往 `AppendRemindMsgToFlow` 中写流，以发出提醒。

注意，**风控终端登录的用户要有管理该投资者的权限**，这样才能收到提醒。

### 预埋强平单的撤回

当发生预埋强平单撤回时，流程与报单类似，由 `RMS_TID_ReqRiskOrderAction` 到 `RMS_TID_ReqParkedOrderAction` 到 `RiskApiMgr`，再报给交易，交易返回一个 response，`RiskApiMgr` 收到后，向流中发送 `RSM_TID_RspParkedOrderAction` package，会调用 `UpdateForceClosingInvByParkedRsp`，真正的实现位于 `AutoRemindHandler.cpp` 中，它会先将强平单状态更新为 `RmsFCOS_Canceled`，表示已经撤销，然后通过 `SearchingQueueingForceCloseOrder` 判断是否可以从 `m_ForceClosingInvSet` 移除用户，最后移除投资者。

* [ ] 个人猜测，点击一次 `强平报单`，尽管会有三个 `ReqRiskOrderInsert` 的数据包，但也许它们的 `FrontID`、`SessionID` 应该是一样的？
* [ ] 为什么要检查强平下单序号 `batchIter->second` 和发过来的数据包转换成 `CRmsForceCloseOrderField`类型后，其 `BatchNo` 字段结果是否相等？

`BatchNo` 和 `ReqForceCloseCalc` 中的 `SequnceNum` 应该是一个意思，都是表示生成的推荐强平单列表左侧的序号。

* [ ] `m_TodayInOutBeforeForceCloseMap` 只会在 `ReqGenForceCloseOrder` 时被更新，所以如果在这个 map 中查找不到对应的的 idx，说明这是手工强平单，是否多此一举了？前面检查过一遍了。
* [ ] 预埋强平单和非预埋强平单用的一套处理逻辑啊？
* [X] fox 预埋强平单是啥？

预埋强平单报单的消息通知流程：风控核心收到 `RMS_TID_ReqRiskOrderInsert` 的 package，如果 `orderField` 的 `IsParked` 字段为 `RmsCB_FALSE`，即 `'1'`，说明这是预埋强平单，则向流中发出 `RMS_TID_ReqParkedOrderInsert` 的 package，`RiskApiMgr` 收到这个 package，`HandleBSMessage` 就会执行 `ReqParkedOrderInsert(inputOrder)`，该函数会执行 `ReqParkedOrderInsert` 来将强平单报给交易前置。

`RMS_TID_ReqInvestorDeposit` 是风控测试用的，会检测出入金，实际生产中不会有这个包。

`HandleTradeTriggeredTask` 中，检测的是 memchange 流中的数据，对于 `FID_UstpOrder` 和 `FIP_UstpTrade`，会执行更新 `UpdateFroceClosingInv` 以更新 `m_ForceClosingInvSet`，预警系统会检查该 set 来判断是否在强平报单之后发生了出入金，如果发生了，则需要提醒。

解流工具：`flowviewer/rmsflowviewer`，执行命令：`../../tools/flowviewer/rmsflowviewer KernelResult.con 0 0 0 > tmp.txt` 可以将流的内容解到 `tmp.txt` 中去。

手动发送出入金消息，并不会触发 `CFtdEngine::HandleNotify` 中的 `RMS_TID_NtfRemindMsg`。

之前是不是把投资者编号弄错了？

出入金的提醒是都会有的吗？  注意 `AppendMsgToFlow` 中的 `pMsgVec`中的 userid 是 risk3 还是 risk1

出入金提醒，最后检查出来发现是 Investor 与 User 的权限问题，本来就有这个检测的逻辑了。

自成交计数问题，gtest

把交易到交易所的路给断掉，

系统发生错误，详细信息请查看日志。

方案 1: 在 RiskServiceXTPLink.cpp 中添加，对于 `case RMS_TID_RspParkedOrderInsert`，

当交易接受到预埋强平单信息后，会发送一个 `RMS_TID_RspParkedOrderInsert` package，`TradeTriggerdServiceXTPLink` 会对该 package 作出响应，已经有了？

用自己的 memchange 流会出现字段检查交易编码字段错误

通过 `dumptool` 可以将内存表 dump 到 `rms_run/rmskernel1/dump` 文件夹下。

## 自成交 gtest 测试用例编写

自成交测试流程 `CheckSelfTradeWarning -> IsSelfTrade -> SaveSelfTradeRecord -> HandleSelfTradeWarning`

`IsSelfTrade` 是判断新来的这笔成交是否会构成自成交，如果构成了自成交，那么才会走到 `SaveSelfTradeRecord`，它会调用 `GetSelfTradeSequenceNo`，该函数中会**计算自成交次数是否增加**，如果增加了，再调用 `HandleSelfTradeWarning`，其中会调用 `CheckIsAddSelfTradeWarningCount`，主要是对集合竞价期间的自成交要特别判断，是否需要预警，

交易编码与成交编号的关系
自成交则成交编号相同、交易编码也相同，一笔成交中，买与卖的成交编号相同。

TDSS 进程负责数据同步，从结算、帐户那里的数据同步到 sync 表。

## 预警

持仓预警流程:

​`AppendGPLWarningToFlow`​ <- `CGradientPosiLimitHandler::GenGPLWarningMsgAndToFlow`​

‍

## 风险通知

​`StandardProtoEngine.cpp`​ 中，`OnRequest`​ 的 `case RMS_TID_SendNotify`​，将 Json 包转换为 Xtp，然后传递给 `riskapimgr`​，`CRiskApiMgr::HandleBSMessage`​会将  `package`​ 转换格式转发给交易，交易处理完成后，发送 Ustp pacakge，`riskapimgr`​通过 CallBack 中的 `CRiskSpi::OnRspSendRiskNotify`​函数处理 Ustp package, 调用 `CRiskApiCallback::OnRspSendRiskNotify`​，该函数调用 `CRiskApiCallback::AppendData`​，会往流中发送 RspSendNotify 的 Xtp package，stdengine 接收到流里的这个包，再发送给网页。

* [ ] 哪个函数调用的 ProtoEngine 的 `OnRequest`​？

‍

短信通知会由 Gbk 转成 utf-8，短信通知似乎不用走后台？

stdfront 接收网页端的消息，是通过流吗？

## 自动风险通知

自动风险通知，可以在 `serviceImpl/tradeTriggered/autoNotify/AutoNotifyHandler.cpp`​ 中找到，具体函数应该是 `CheckPositionInitNotify()`​，当前该函数只会检查投资者是否由于期权即将到期而需要自动通知。

‍

## XTP Package 结构

Package 总的来说，包含 header 与 content 两部分，header 中的 `Chain`​ 字段和 `Tid`​ 字段是比较关键的两个字段，content 中是一系列的 field，即结构体，可能一个，可能多个，受 `XTPMisc.xml`​ 中的 `minOccur`​ 和 `maxOccur`​ 限制，每个 Tid 的 package 可能包含哪些 field，也由 `XTPMisc.xml`​ 决定。

* 以 Tid 为 0x8202 的 package 为例：其 `RmsRspInfo`​ field 的 `maxOccur`​ 为 1，说明该 filed 在这个 package 中只会出现一次，因此使用 `XTP_GET_SINGLE_FIELD`​ 来根据 field 的 `fid`​ 来获取该 filed 的内容；

* 而对于 Tid 为 0x856e 的 package 的 `RmsRcamsCombRule`​ field 来说，其 `maxOccur`​ 为 `*`​，说明可能出现多次，通过以下手段以获取这些 field 的内容：

```cpp
CRmsRcamsCombRuleField field;
while (!it.IsEnd()) {
	it.Retrieve(&field);
	it.Next();
}
```

而 header 中的 `Chain`​ 由 `C`​ 和 `L`​ 两种值，`C`​ 表示 continue，`L`​ 表示 last，可能一个 package 放不下总共需要的这些数据，那么就需要连续读取同一 Tid 的多个 package，直到 header 的 `Chain`​ 字段为 `L`​，说明这是这一组数据的最后一个 package 了，这些 package 在流中不一定是连续的，本身读取的时候，只会关注同一 tid 的 package，从 `C`​ 一直读取到 `L`​ 结束。

‍

一个是解决部署的后台连上 C 端报 502 error 的问题，主要是修复 C 端的两个缺陷，一个是批量强平强平单列表只能生成一条，还有一个是通知内容乱码的问题，这两个问题都已经定位到了，但是要解决的话还需要前端那边调整一下；

‍

‍

‍

‍

‍

‍

‍

‍
