## Cocoapods
`pod install`会先去更新Cocoapods master分支下所有的podspec信息，也可以禁用更新：

```
pod install --no-repo-update
```

`pod install`工作流程：
1.判断 Podfile.lock 是否存在，如果不存在，按照 Podfile 中指定的版本安装
2.如果 Podfile.lock 存在，检查 Podfile 中每一个 Pod 在 Podfile.lock 中是否存在
3.如果存在， 则忽略 Podfile 中的配置，使用 Podfile.lock 中的配置(实际上就是什么都不做)
4.如果不存在，则使用 Podfile 中的配置，并写入 Podfile.lock 中

`pod update`会忽略 Podfile.lock 文件，完全使用 Podfile 中的配置，并且更新 Podfile.lock。