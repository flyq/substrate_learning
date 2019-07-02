# 源码学习二 node-template

 ## `main.rs` 
 启动入口

##  `cli.rs`
`run`函数，然后调用 `core` 里面的 `parse_and_execute`。还包括一些运行退出内容。

 ## `service.rs` 
 服务，包括本机执行实例： `Executor`，`dispatch`，`native_version`。还有 `Service` 工厂：`construct_service_factory`，
 里面包括 `Block`，`RuntimeApi`， `NetworkProtocol`， `FullTransactionPoolApi`， `LightTransactionPoolApi`，  `Genesis`， `Configuration`， `FullService`， `AuthoritySetup`， `LightService`，`FullImportQueue`， `LightImportQueue`， `SelectChain`， `FinalityProofProvider`。 这些大部分需要调用 `core` 里面的实现

 ## `chain_spec.rs`
