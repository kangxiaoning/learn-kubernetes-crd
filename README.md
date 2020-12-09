
# 通过code-generator生成kubernetes CRD代码

这里以添加一个 kind 为 `Machine` ，group 为 `crd.alpha.io` ，类型为 `v1` 的 CRD 为例。

该 CRD 的定义如下。

```bash
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: machines.crd.alpha.io
spec:
  group: crd.alpha.io
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cpu:
                  type: integer
                memory:
                  type: integer
                storage:
                  type: integer
  names:
    kind: Machine
    plural: machines
  scope: Namespaced
```

该 CRD 的使用如下。

```bash
apiVersion: crd.alpha.io/v1
kind: Machine
metadata:
  name: example-alpha
spec:
  cpu: 2
  memory: 4
  storage: 10
```

## 构建项目

### 初始化项目

环境信息

| component  | version |
| :------    | :------ |
| go         | 1.15.2  |
| kubernetes | v0.19.4 |



```bash

cd $GOPATH/src/github.com/kangxiaoning
mkdir learn-kubernetes-crd
cd learn-kubernetes-crd
mkdir -p pkg/apis/crd/v1
go mod init github.com/kangxiaoning/learn-kubernetes-crd

```

### 定义资源类型

#### pkg/apis/crd/register.go

`pkg/apis/crd/register.go`，保存后面要使用的全局变量。

```go
package crd

const (
	GroupName = "crd.alpha.io"
	Version   = "v1"
)
```

#### pkg/apis/crd/v1/doc.go

`pkg/apis/crd/v1/doc.go`，定义 Golbal Tags，利用 `+<tag_name>[=value]` 的注释格式控制全局代码生成，详细使用参考[Kubernetes Deep Dive: Code Generation for CustomResources](https://www.openshift.com/blog/kubernetes-deep-dive-code-generation-customresources)。

```go
// +k8s:deepcopy-gen=package

// +groupName=crd.alpha.io
package v1
```

#### pkg/apis/crd/v1/types.go

`pkg/apis/crd/v1/types.go`，定义 `Machine` 资源类型。
```go 
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Machine describes a Machine resource
type Machine struct {
	// TypeMeta is the metadata for the resource, like kind and apiversion
	metav1.TypeMeta `json:",inline"`
	// ObjectMeta contains the metadata for the particular object, including
	// things like...
	//  - name
	//  - namespace
	//  - self link
	//  - labels
	//  - ... etc ...
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec MachineSpec `json:"spec"`
}

// MachineSpec is the spec for a Machine resource
type MachineSpec struct {
	// this is where you would put your custom resource data
	Cpu     int `json:"cpu"`
	Memory  int `json:"gateway"`
	Storage int `json:"storage"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// MachineList is a list of Machine resources
type MachineList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []Machine `json:"items"`
}
```

#### pkg/apis/crd/v1/register.go

`pkg/apis/crd/v1/register.go`，注册资源类型以便生成的 client 代码能关联到该类型。

```go 
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"

	"github.com/kangxiaoning/learn-kubernetes-crd/pkg/apis/crd"
)

// GroupVersion is the identifier for the API which includes
// the name of the group and the version of the API
var SchemeGroupVersion = schema.GroupVersion{
	Group:   crd.GroupName,
	Version: crd.Version,
}

// create a SchemeBuilder which uses functions to add types to
// the scheme
var (
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme   = SchemeBuilder.AddToScheme
)

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

// Kind takes an unqualified kind and returns back a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// addKnownTypes adds our types to the API scheme by registering
// Machine and MachineList
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(
		SchemeGroupVersion,
		&Machine{},
		&MachineList{},
	)

	// register the type in the scheme
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```

### 配置code-generator

在生成代码前需要做些配置。

```bash
cd $GOPATH/src/github.com/kangxiaoning/learn-kubernetes-crd
mkdir hack
```

#### hack/tools.go

`hack/tools.go`，用来依赖 `code-generator`，以便可以使用该包。

```go 
// +build tools

/*
   Copyright 2019 The Kubernetes Authors.

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
*/

// This package imports things required by build scripts, to force `go mod` to see them as dependencies
package tools

import _ "k8s.io/code-generator"
```

#### hack/boilerplate.go.txt

`hack/boilerplate.go.txt`，构建 API 时的文件头，生成代码时需要这个文件。

```
/*
Copyright The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/
```

#### hack/update-codegen.sh

```bash
#!/usr/bin/env bash

# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set -o errexit
set -o nounset
set -o pipefail

# generate the code with:
# --output-base    because this script should also be able to run inside the vendor dir of
#                  k8s.io/kubernetes. The output-base is needed for the generators to output into the vendor dir
#                  instead of the $GOPATH directly. For normal projects this can be dropped.
ROOT_PACKAGE="github.com/kangxiaoning/learn-kubernetes-crd"
CUSTOM_RESOURCE_NAME="crd"
CUSTOM_RESOURCE_VERSION="v1"

chmod +x ../vendor/k8s.io/code-generator/generate-groups.sh

../vendor/k8s.io/code-generator/generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION" \
  --go-header-file $(pwd)/boilerplate.go.txt \

```

### 生成代码 

执行如下命令生成代码。

```
cd $GOPATH/src/github.com/kangxiaoning/learn-kubernetes-crd
go mod vendor
chmod +x hack/update-codegen.sh
cd hack && ./update-codegen.sh

```

生成代码过程及内容如下。

```bash


➜  learn-kubernetes-crd cd hack
➜  hack ./update-codegen.sh
Generating deepcopy funcs
Generating clientset for crd:v1 at github.com/kangxiaoning/learn-kubernetes-crd/pkg/client/clientset
Generating listers for crd:v1 at github.com/kangxiaoning/learn-kubernetes-crd/pkg/client/listers
Generating informers for crd:v1 at github.com/kangxiaoning/learn-kubernetes-crd/pkg/client/informers
➜  hack


➜  hack cd ..
➜  learn-kubernetes-crd tree pkg/
pkg/
├── apis
│   └── crd
│       ├── register.go
│       └── v1
│           ├── doc.go
│           ├── register.go
│           ├── types.go
│           └── zz_generated.deepcopy.go
└── client
    ├── clientset
    │   └── versioned
    │       ├── clientset.go
    │       ├── doc.go
    │       ├── fake
    │       │   ├── clientset_generated.go
    │       │   ├── doc.go
    │       │   └── register.go
    │       ├── scheme
    │       │   ├── doc.go
    │       │   └── register.go
    │       └── typed
    │           └── crd
    │               └── v1
    │                   ├── crd_client.go
    │                   ├── doc.go
    │                   ├── fake
    │                   │   ├── doc.go
    │                   │   ├── fake_crd_client.go
    │                   │   └── fake_machine.go
    │                   ├── generated_expansion.go
    │                   └── machine.go
    ├── informers
    │   └── externalversions
    │       ├── crd
    │       │   ├── interface.go
    │       │   └── v1
    │       │       ├── interface.go
    │       │       └── machine.go
    │       ├── factory.go
    │       ├── generic.go
    │       └── internalinterfaces
    │           └── factory_interfaces.go
    └── listers
        └── crd
            └── v1
                ├── expansion_generated.go
                └── machine.go

20 directories, 27 files
➜  learn-kubernetes-crd

```

如上便是生成代码的整体过程，需要注意的是用 go mod 机制后，需要让项目依赖 code-generator，以及需要配置 code-generator 需要的文件。

# 编写custom controller


上篇文章介绍了通过code-generator生成CRD的代码，即让kubernetes支持自定义资源，接下来为我们为这个CRD编写controller。

## controller工作原理

引用社区一张图，描述了controller的工作流程。

![client-go controller interaction](https://github.com/kubernetes/sample-controller/blob/master/docs/images/client-go-controller-interaction.jpeg)

## 编写controller代码

借助前面生成的代码，基本只需要编写业务逻辑相关的代码了，具体包括三部分。

1. signals，shutdown gracefully，直接copy[社区示例](https://github.com/kubernetes/sample-controller/tree/master/pkg/signals)即可。
2. controller.go，具体逻辑实现，包括注册handler以便处理对应事件，从informer本地cache获取event，启动Control Loop针对不同的事件执行具体的业务逻辑。
3. main.go，创建informer及client，构造controller，最后启动informer及controller.Run。

本示例代码在[github](https://github.com/kangxiaoning/learn-kubernetes-crd)可供参考。


通过如下命令编译。

```bash
go mod vendor
go build -o machine-controller .
```

## controller流程验证

验证过程如下。

1. 运行controller

由于此时还没有对应资源，watch会失败。

```bash
➜  learn-kubernetes-crd git:(main) ./machine-controller  -kubeconfig=$HOME/.kube/config
I1209 16:34:47.229297   23302 controller.go:103] Setting up event handlers
I1209 16:34:47.229562   23302 controller.go:133] Starting Machine controller
I1209 16:34:47.229638   23302 controller.go:136] Waiting for informer caches to sync
E1209 16:34:47.258218   23302 reflector.go:127] github.com/kangxiaoning/learn-kubernetes-crd/pkg/client/informers/externalversions/factory.go:117: Failed to watch *v1.Machine: failed to list *v1.Machine: the server could not find the requested resource (get machines.crd.alpha.io)
E1209 16:34:48.546412   23302 reflector.go:127] github.com/kangxiaoning/learn-kubernetes-crd/pkg/client/informers/externalversions/factory.go:117: Failed to watch *v1.Machine: failed to list *v1.Machine: the server could not find the requested resource (get machines.crd.alpha.io)
```

2. 创建CRD

```bash

➜  learn-kubernetes-crd git:(main) kubectl apply -f crd/alpha-machine.yaml
customresourcedefinition.apiextensions.k8s.io/machines.crd.alpha.io created
➜  learn-kubernetes-crd git:(main)
```

controller watch到CRD，不再报错，启动了worker

```
I1209 16:36:07.728882   23302 controller.go:141] Starting workers
I1209 16:36:07.728931   23302 controller.go:147] Started workers
```

3. 创建machine实例

```bash
➜  learn-kubernetes-crd git:(main) kubectl apply -f example/example-machine.yaml
machine.crd.alpha.io/example-alpha created
➜  learn-kubernetes-crd git:(main)
```

controller监听到add事件，执行了add事件对应的handler

```bash
I1209 16:37:49.982159   23302 controller.go:254] [Alpha] Try to process machine: &v1.Machine{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"example-alpha", GenerateName:"", Namespace:"default", SelfLink:"/apis/crd.alpha.io/v1/namespaces/default/machines/example-alpha", UID:"0386f928-2e59-4699-bba9-ee9a6ae66aec", ResourceVersion:"109312", Generation:1, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:63743099869, loc:(*time.Location)(0x2aaa6c0)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"crd.alpha.io/v1\",\"kind\":\"Machine\",\"metadata\":{\"annotations\":{},\"name\":\"example-alpha\",\"namespace\":\"default\"},\"spec\":{\"cpu\":2,\"memory\":4,\"storage\":10}}\n"}, OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry{v1.ManagedFieldsEntry{Manager:"kubectl-client-side-apply", Operation:"Update", APIVersion:"crd.alpha.io/v1", Time:(*v1.Time)(0xc000610580), FieldsType:"FieldsV1", FieldsV1:(*v1.FieldsV1)(0xc000610560)}}}, Spec:v1.MachineSpec{Cpu:2, Memory:0, Storage:10}} ...
I1209 16:37:49.982406   23302 controller.go:207] Successfully synced 'default/example-alpha'
I1209 16:37:49.982480   23302 event.go:282] Event(v1.ObjectReference{Kind:"Machine", Namespace:"default", Name:"example-alpha", UID:"0386f928-2e59-4699-bba9-ee9a6ae66aec", APIVersion:"crd.alpha.io/v1", ResourceVersion:"109312", FieldPath:""}): type: 'Normal' reason: 'Synced' Machine synced successfully
```

3. 更新machine实例

此处将cpu从2修改为4.

```bash
➜  learn-kubernetes-crd git:(main) vim example/example-machine.yaml
➜  learn-kubernetes-crd git:(main) ✗ kubectl apply -f example/example-machine.yaml
machine.crd.alpha.io/example-alpha configured
➜  learn-kubernetes-crd git:(main) ✗
```

controller监听到update事件，执行了相应的handler

```bash
I1209 16:40:08.962490   23302 controller.go:254] [Alpha] Try to process machine: &v1.Machine{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"example-alpha", GenerateName:"", Namespace:"default", SelfLink:"/apis/crd.alpha.io/v1/namespaces/default/machines/example-alpha", UID:"0386f928-2e59-4699-bba9-ee9a6ae66aec", ResourceVersion:"109618", Generation:2, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:63743099869, loc:(*time.Location)(0x2aaa6c0)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"crd.alpha.io/v1\",\"kind\":\"Machine\",\"metadata\":{\"annotations\":{},\"name\":\"example-alpha\",\"namespace\":\"default\"},\"spec\":{\"cpu\":4,\"memory\":4,\"storage\":10}}\n"}, OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry{v1.ManagedFieldsEntry{Manager:"kubectl-client-side-apply", Operation:"Update", APIVersion:"crd.alpha.io/v1", Time:(*v1.Time)(0xc0001822a0), FieldsType:"FieldsV1", FieldsV1:(*v1.FieldsV1)(0xc000182280)}}}, Spec:v1.MachineSpec{Cpu:4, Memory:0, Storage:10}} ...
I1209 16:40:08.962661   23302 controller.go:207] Successfully synced 'default/example-alpha'
I1209 16:40:08.962689   23302 event.go:282] Event(v1.ObjectReference{Kind:"Machine", Namespace:"default", Name:"example-alpha", UID:"0386f928-2e59-4699-bba9-ee9a6ae66aec", APIVersion:"crd.alpha.io/v1", ResourceVersion:"109618", FieldPath:""}): type: 'Normal' reason: 'Synced' Machine synced successfully
```

4. 删除machine实例

```bash
➜  learn-kubernetes-crd git:(main) ✗ kubectl delete -f example/example-machine.yaml
machine.crd.alpha.io "example-alpha" deleted
➜  learn-kubernetes-crd git:(main) ✗
```

controller监听到delete事件，执行了相应的handler

```bash
I1209 16:41:55.310657   23302 controller.go:167] get "default/example-alpha"
W1209 16:41:55.310696   23302 controller.go:237] Machine: default/example-alpha does not exist in local cache, will delete it from Alpha ...
I1209 16:41:55.310702   23302 controller.go:240] [Alpha] Deleting machine: default/example-alpha ...
I1209 16:41:55.310708   23302 controller.go:207] Successfully synced 'default/example-alpha'
```

如上是编写custom controller的大概步骤，通过工作原理结合代码实现，可以更深刻的理解kubernetes Control Loop的工作原理和设计理念，在kubernetes核心资源无法满足需求时，可以通过CRD+Controller或operator来满足。
