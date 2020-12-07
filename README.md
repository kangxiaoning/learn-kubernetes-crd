这里以添加一个 kind 为 `Machine` ，group 为 `crd.alpha.io` ，类型为 `v1` 的 CRD 为例。

该 CRD 的定义如下。

```bash
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: machine.crd.alpha.io
spec:
  group: crd.alpha.io
  version: v1
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

../vendor/k8s.io/code-generator/generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION" \
  --go-header-file $(pwd)/boilerplate.go.txt \

```

### 生成代码 

执行如下命令生成代码。

```
cd $GOPATH/src/github.com/kangxiaoning/learn-kubernetes-crd
go mod vendor
chmod +x vendor/k8s.io/code-generator/generate-groups.sh
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
