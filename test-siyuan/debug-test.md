---
title: debug
slug: debug-test
url: /post/debug-test.html
date: '2024-09-03 09:31:46+08:00'
lastmod: '2024-09-05 18:16:08+08:00'
toc: true
tags:
  - test
  - debug
isCJKLanguage: true
keywords: test,debug
---

# debug

# 缺陷修复记录

## 调试的前期准备

修改环境变量，将 `startfile` 脚本中设置环境变量的部分复制到 `.bashrc` 中去。

对于单个程序，可以进入 `FIP_RISK/02Backend/target/${prog_name}` 目录下，通过执行以下命令编译生成可执行文件，以 `rmskernel` 为例。

```sh
amake rsmkernel.prj
make clean
make pump
make -j4
```

调试时，在 `make pump` 这一步之后，修改 `make file`，将 `APPEND_CPPFLAGS` 从 `O3` 修改为 `O0`，再进行编译。

调试打好断点运行时，需要输入 `r 1` 而不是 `r`。

## 强平试算结果异常问题

### 问题表现

在风控管理端的单客户强平功能中，勾选多个交易项然后点击 `强平单生成`，对于生成后的强平单，分别对单项交易点击 `强平试算`，只有序号为 1 的项的试算结果是正确的，其他项的结果的释放保证金为 0。

> 事实上，后期才确定结果的正确性与交易的左侧序号有关，我之前还以为是特定的交易结果不对。

### Debug 过程

在 `CKernelRiskReactor::HandleMessage` 处打上断点，然后执行 `r 1`，然后再删除该断点，再在 `CRiskServiceImpl::PreReqGenForceCloseOrder` 处打上断点，这样处理的好处是可以直接跳过 Redo 的过程（`Redo` 也会调用 `PreReqForceCloseOrder`）。

以投资者 `801000851` 为例，首先，对于 `强平单生成按钮`，点击之后会执行 x 次 `PreReqGenForceCloseOrder`，将待处理的交易都添加到一个集合中，然后执行一次 `ReqGenForceCloseOrder`，处理集合中的这些交易，生成结果（即“推荐强平单”）。

> x 为勾选的要生成强平单的交易数量。

在点击 `强平试算` 按钮时，也会执行 x 次 `PreReqForceCloseCalc`，将待试算的单子都添加到一个集合中去，然后执行一次 `ReqGenForceCloseOrder`，对这些单子调用试算核心进行试算，得出试算结果。

我直接勾选了前三个交易以生成“强平推荐单”，其中第二个单子为郑商所的合约，而在计算要释放的保证金的步骤是这样的，首先计算各个投资者在交易所的初始保证金之和（后简称交易所保证金），即 `moneyField.InitMargin`，对于同一个投资者的不同强平单而言，该值都是一样的。

生成强平单时，我们选择的算法为“按照预结算资金”处理，故 `pReqField->Algorithm` 不为 `RmsFCA_ByRealTimeFunds`，具体算法对应的枚举值，可以在 `UFEntity.xml` 和 `UFDataType.xml` 中查找，然后在第二次执行 `m_pForceCloseBase->SettleCalc` 时，会更新求得 `moneyField.Margin`，这是通过伪造成交记录后，计算各个投资者在各个交易所的保证金之和得到的，具体内容可以查看 `ForceCloseByPrice.cpp` 中的 `SettleCalc` 函数。

`SettleCalc` 函数中，可以看到会遍历 `m_ExchangeVec` 进行处理，即每个交易所都处理一次，考虑到我用来测试的第二个单子是郑商所的，那么我只要关注投资者在郑商所的保证金变化即可，这里重点关注 `GetResult`，将会输出交易所 ID 与 `ExTradeMargin` 的日志的输出级别从 `LOG_EVENT` 修改为 `LOG_INFO`，这样相关日志就会写入到 `rms_run/rmskernel1/bin/Syslog.log` 中，查看日志，可以发现在执行过 `StartSettle` 之后，郑商所的保证金仍为 89852，根据益伟哥的提示，本次 bug 大概率出现在 `PrepareBasicData` 或者清算核心中，这里先检查比较方便检查的 `PrepareBasicData`。

在 `PrepareBaicData` 中进行单步调试，对于其中调用的函数，也再进入到其函数内进行单步调试，故进入 `GetSettlePosiInfo` 中进行单步调试，该函数大概是用于获取结算后的持仓信息的，单步调试时发现一个明显的问题，`m_FakeTradeMap.find(m_ExchangeID) == m_FakeTradeMap.end()` 成立了。

`GetSettlePosiInfo` 函数直接返回，这说明，从 `m_FakeTradeMap` 这个名字可以看出，应该是与伪造成交记录有关的，故查看 `FakeForceCloseTrade` 函数，发现 `pFCOrderVec->size() == 0` 成立，即 `newFCOrderVec->size()` 为 0，故直接定位到 `ReqForceCloseCalc` 函数中修改更新 `newFCOrderVec` 的部分，通过 gdb 可以发现，`reqVecIter->SequenceNum` 与 `reqMapIter->second.size()` 均为 1，故 `newFCOrderVec` 没有被修改，因此，我们找到了问题所在。

`SequenceNum` 应该是我勾选的推荐强平单左边的序号再减一。

从 `reqMapIter = m_ReqForceCloseCalMap.find(idx)` 这一定义可以看出，`reqMapIter` 应该是我们勾选的要进行试算的强平单的集合的迭代器，因此，要修改这个问题很简单，删除 `if` 判断即可，这个 if 判断本来就没啥意义，因为更新 `newFCOrderVec` 就是用 `reqMapIter` 来迭代的。

这里，为了保险起见，我们在注意一下 `tmpStruct = (orderMapIter->second)[reqVecIter->SequenceNum.getValue]`。

可以看到 `orderMapIter` 是 `m_InvestorRspGenFCOrderMap` 的迭代器，而 `m_InvestorRspFCOrderMap` 只在 `InitRspVec` 函数中被调用，这里我们查看 `InitRspVec` 函数被谁调用了，发现是 `ReqGenForceCloseOrder` 函数中，执行了 `InitRspVec(idx, pRspVec)`，`InitRspVec` 的参数其中有一个是 `vector<A> *& p`，即 vector 的指针的引用，在 `InitRspVec` 函数中，`m_InvestorRspGenFCOrderMap` 的值是一个 `vector<A>`，而这个值的指针被赋给 `pRspVec`，执行完 `InitRspVec` 之后，就能通过修改 `pRspVec` 指向的地址中的内容修改 `m_InvestorRspGenFCOrderMap` 了，从变量定义可知，该 map 是用于存放推荐强平后的强平单，即点击“强平单生成”之后得到的推荐强平单列表。

### 总结与反思

这次 bug 的定位与修复耗时大约两个人日，比较久，一是对业务以及业务代码还是不太了解，比如强平试算、生成强平单，是按照什么标准来进行的，管理端的各个选项分别对应代码的什么；二是对变量的功能以及什么时候被更新不熟悉；三是 gdb 太久没用了，不熟悉，还有就是代码的一些特殊性，比如打好断点之后运行函数应该执行 `r 1` 而不是 `r`，`start_file` 脚本会自动添加环境变量，所以自己应该在 `.bashrc` 中添加环境变量，部署风控后台也不熟悉。

总之，要多熟悉 gdb 的使用，操作风控管理端时，要注意按下按钮会执行代码中的哪个或者哪些函数，输出的结果与代码中哪些变量有关，还有就是代码中虚函数接口用得很多，要注意函数实现到底在哪里。

> 代码中虚函数接口用得很多，要注意到底调用的哪个子类的函数。

‍

## B 端强平单生成列表只有一条

* [ ] 伟正注释掉了 `StandardProtoEngine.cpp`​ 的 `OnRequest`​中 `case RMS_TID_ReqForceCloseOrder`​ 和 `case RMS_TID_RspForceCloseOrder`​ 两个部分，为什么？

在 `HandleResponese`​ 处，添加 `TransFieldtoBSField`​，该模板函数调用 `CopyFieldToBS`​，从而将 `CRmsNotifyMsgInputField`​ 中的 `Content`​ 的编码由 gbk 修改为 utf-8。

## B 端风险通知乱码

‍
