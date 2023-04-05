# 我的React源码分析学习笔记

## 笔记记录形式

综合使用了ProcessOn的画图、drawio画图、XMind、Typora等平台的记录形式，最后感觉是drawio和Typera两种形式结合更容易表达自己对React源码的链路分析记录。所以目录中的主体记录形式是drawio文件和markdown文件。

## markdown文件

markdown文件以代码段形式摘录对应React官方GitHub项目的源文件代码，版本是tag v18.2.0。
摘录的代码会对源码中的一些检查校验提示等非主体逻辑去除，剩下主体需要关注的代码内容，着重分析思考

## 配合 图解React源码

图解React: <https://7kms.github.io/react-illustration-series/>

Github: <https://github.com/7kms/react-illustration-series>

图解React对React源码分析的挺清楚和详细，但其主体基于分析的React代码版本是17.0.2
作为参考，分析前尽量自己先对React的api进行代码debug调试(v18.2.0)，追踪代码链路和调用栈，先自己对这些代码调用有自己的认识和见解(一般自己会追踪1-2遍，并先记录下自己的对主体核心部分内容的记录，再配合图解里的文章进行比对思考)
