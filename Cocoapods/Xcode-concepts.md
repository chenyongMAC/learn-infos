参考地址：https://developer.apple.com/library/content/featuredarticles/XcodeConcepts/Concept-Targets.html

## Target
target明确了一个产品的构建过程，明确了在一个project或workspace中用一系列文件构建一个产品所包含的说明。
Projects可以包含一到多个targets，每一个targets都能用于产生一个产品，但一个产品只能使用一个targettarget从project中继承了build settings，你也可以重写它的一些配置。

target的创建可以依赖于别的target。当一个workspace钟包含多个target时，xcode可以发现他们的依赖关系，并正确的编译它们（主要通过build setting中的配置确定）。


## Project
包含了所有的源文件，资源文件和构建一个或者多个product的信息。project利用他们去编译我们所需的product，也帮我们组织它们之间的关系。

## Workspace
workspace是Xcode的一种文件，用来管理工程和里面的文件，一个workspace可以包含若干个工程，甚至可以添加任何你想添加的文件。workspace提供了工程和工程里面的target之间隐式和显式依赖关系，用来管理和组织工程里面的所有文件。

## Scheme
一个 Xcode scheme 定义了编译集合中的若干 target，编译时的一些设置以及要执行的测试集合。
即我们cmd+R之前会先选择一个“运行目标”，这个运行目标就是scheme。