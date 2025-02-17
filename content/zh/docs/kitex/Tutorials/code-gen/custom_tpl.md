---
title: "自定义脚手架模板"
date: 2023-03-09
weight: 7
description: >
---

Kitex 支持了自定义模板功能，如果默认的模板不能够满足大家的需求，大家可以使用 Kitex 的自定义模板功能。目前支持了单文件渲染和根据 methodInfo 循环渲染。

## 如何使用

1.  模板使用的数据为 PackageInfo，认为这部分内包含所有的元数据，如 methodInfo 等，用户只需要传递模板文件即可，模板内的数据为 PackageInfo 数据。
1.  模板渲染只能渲染单个文件内容，如果涉及到按照 methods 分文件渲染的话，需要在代码中控制。
1.  模板文件通过 yaml 文件夹传递，通过 `--template-dir` 命令行参数指定。因为所有模板写到一个文件中会导致该文件巨大，所以改为指定文件夹。
1.  文件夹内的 `extensions.yaml` 为特定文件，该文件的内容为[扩展 Service 代码](https://www.cloudwego.io/zh/docs/kitex/tutorials/code-gen/template_extension/)的配置文件。如果该文件存在的话，则不用再传递 `template-extension` 参数。
1.  更新时，目前只支持覆盖、跳过和根据 methods 增加文件三种，支持在一个文件当中 append。增加文件就是比如我按照 methods 循环渲染，如果用户新增加了 methods，那就会新增加文件。
1.  Kitex 代码生成分成两部分，kitex_gen 和 mainPkg(剩下的 main.go、handler.go )等等，kitex_gen 无论采用何种生成都不会改变；mainPkg 和 custom layout 只能二选一，如果制定了 custom layout 就不会再生成 mainPkg。

## 使用场景
当默认的脚手架模板不能够满足用户的需求，比如想要生成 MVC Layout、统一进行错误处理等。

## 使用方式

```console
kitex -module ${module_name} -template-dir ${template dir_path} idl/hello.thrift
```

## Yaml 文件配置

```yaml
path: /a/main.go # 生成文件的路径及文件名，这会在项目根目录下创建 a 文件夹，并在文件夹内生成 main.go 文件
update_behavior:
    type: skip / cover / append # 指定更新行为，如果 loop_method 为true，则不支持 append。默认是 skip
    key: Test{{.Name}} # 函数名
    append_tpl: # 更新的内容模板
    import_tpl: # 新增的 import 内容，是一个 list，可以通过模版渲染
body: template content # 模板内容
--------------------------------
path: /handler/{{ .Name }}.go 
update_is_skip: true # 更新时不跳过该文件，由于指定了 loopfield，所以更新时会增加文件
loop_method: true
body: ... # 模板内容
```

示例 tpl：

https://github.com/cloudwego/cwgo/tree/main/tpl/kitex


## 附录

### PackageInfo 结构体常用内容

```go
type PackageInfo struct {
   Namespace    string            // idl namespace，pb 下建议不要使用
   Dependencies map[string]string // package name => import path, used for searching imports
   *ServiceInfo                   // the target service

   Codec            string
   NoFastAPI        bool
   Version          string
   RealServiceName  string
   Imports          map[string]map[string]bool 
   ExternalKitexGen string
   Features         []feature
   FrugalPretouch   bool
   Module           string // go module 名称
}

type ServiceInfo struct {
   PkgInfo
   ServiceName     string
   RawServiceName  string
   ServiceTypeName func() string
   Base            *ServiceInfo
   Methods         []*MethodInfo
   CombineServices []*ServiceInfo
   HasStreaming    bool
}

type PkgInfo struct {
   PkgName    string // namespace 最后一段
   PkgRefName string
   ImportPath string // 这个方法的 req 和 resp 的 import path
}

type MethodInfo struct {
   PkgInfo
   ServiceName            string // 这个 service 的 name
   Name                   string // 这个 method 的 name
   RawName                string // 同上
   Oneway                 bool
   Void                   bool
   Args                   []*Parameter // 入参信息，包括入参名称、import 路径、类型
   Resp                   *Parameter // 出参，包括入参名称、import 路径、类型
   Exceptions             []*Parameter
   ArgStructName          string
   ResStructName          string
   IsResponseNeedRedirect bool // int -> int*
   GenArgResultStruct     bool
   ClientStreaming        bool
   ServerStreaming        bool
}

// Parameter .
type Parameter struct {
   Deps    []PkgInfo 
   Name    string  
   RawName string // StructB
   Type    string // *PkgA.StructB
}
```
