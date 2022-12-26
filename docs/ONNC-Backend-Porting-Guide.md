# ONNC Backend Developer Guide

릴리스 PDF: [ONNC-Backend-Developer-Guide.pdf](https://github.com/ONNC/onnc/wiki/files/1.0.0/ONNC-Backend-Porting-Guide.pdf)

이 애플리케이션 노트는 대상 하드웨어에 대한 ONNC 백엔드를 생성하는 방법을 보여줍니다. 이 문서는 ONNC 커뮤니티 Docker 이미지 v1.0을 준수합니다. Docker 허브(https://hub.docker.com/r/onnc/onnc-community/)에서 Docker 이미지를 다운로드할 수 있습니다.

## 1. Introduction

ONNC는 딥 러닝 가속기(DLA)를 대상으로 하는 오픈 소스, 모듈식, 재사용 가능한 컴파일러 알고리즘 및 도구 체인 모음입니다. ONNC는 처음부터 ONNX 중간 표현(IR)을 독점 DLA 코드로 변환하기 위해 구축되었습니다. 소프트웨어 아키텍처 디자인은 이식성과 재사용성을 강조하여 대상 변경을 단순화합니다. 그림 1은 ONNC 소프트웨어 스택의 최상위 블록 다이어그램을 보여줍니다. 소프트웨어 스택은 ONNX 계산 그래프 모델 가져오기에서 해당 하드웨어 바이너리 내보내기까지의 기능 블록을 보여줍니다. LLVM 백엔드를 활용하는 것 외에도 ONNC는 ONNX IR에 대한 일대일 매핑이 있는 IR(중간 표현)인 ONNC IR을 정의하여 독점 DLA가 ONNX 모델을 실행할 수 있는 또 다른 빠른 경로를 제공합니다. 딥 러닝 시스템의 다른 두 가지 인기 있는 컴파일 프레임워크인 TVM과 Glow는 LLVM 백엔드 위에 소프트웨어 스택을 구축했습니다. LLVM의 중간 표현은 하드웨어 연산자에 매핑하는 동안 ONNC IR보다 더 세분화됩니다. 컨볼루션과 같은 거친 연산자로 구축된 가속기의 경우 LLVM 백엔드를 해킹하려면 더 많은 포팅 노력이 필요합니다. Nvidia의 [NVDLA](http://nvdla.org/) 및 Bitman의 [Sophon BM168X 시리즈](https://sophon.ai/product/introduce/bm1682.html)와 같은 많은 DLA 설계는 LLVM보다 거친 연산자를 선호합니다. 운영자. 이러한 경우 ONNC는 자체 Vanilla 백엔드를 사용하여 ONNX 모델을 대상 바이너리로 변환하는 보다 간단한 방법을 제공합니다. 따라서 컴파일러를 새 하드웨어로 포팅하는 속도가 빨라집니다. 빠른 포팅을 위해 사용자는 Vanilla 백엔드를 템플릿으로 복사하고 최소한 두 개의 소프트웨어 파이프를 재정의하고 선택적 최적화 패스를 추가하기만 하면 프레임워크가 나머지 작업을 매력처럼 처리합니다.

![](https://github.com/ONNC/onnc/wiki/files/1.2.0/onnc-software-architecture-diagram.png)

**Figure 1. ONNC Software Architecture Diagram**
 

## 2. Quick Start

ONNC는 백엔드 "스켈레톤"을 빠르게 파생시키는 스크립트를 제공합니다. 이 스켈레톤은 유용하지 않지만 백엔드의 실행 가능한 최소 코드를 보여줍니다. 스켈레톤을 기반으로 백엔드 개발을 시작할 수 있습니다. 이 섹션에서는 해당 스크립트를 실행하여 스켈레톤을 생성하는 방법과 생성된 백엔드를 컴파일하는 방법을 소개합니다.


### 2.1. Getting the Docker Image

ONNC 구축 프로세스를 용이하게 하기 위해 Docker 이미지를 제공합니다. 진행하기 전에 Docker 프로그램(http://www.docker.com)이 컴퓨터에 설치되어 있는지 확인하십시오.

그런 다음 다음 셸 명령으로 ONNC Docker 이미지를 가져옵니다.
```console
$ docker pull onnc/onnc-community
```

### 2.2. Getting the ONNC source codes

The latest ONNC source code is available on GitHub. Follow these commands to download the source code.

```console
$ git clone https://github.com/ONNC/onnc.git
```

### 2.3. Creating a new backend

ONNC 커뮤니티 Docker에는 새로운 백엔드 생성을 위한 스크립트가 있습니다. 해당 스크립트를 실행하려면 ONNC Docker 내에서 Ubuntu 프롬프트를 입력하십시오.

```console
// Use the interactive mode to enter the Docker prompt.
$ docker run -ti --rm -v <onnc_source_dir>:/onnc/onnc onnc/onnc-community
```

`-v <onnc_source_dir>:/onnc/onnc` 옵션은 ONNC 소스 코드를 Docker 컨테이너에 마운트하는 것을 의미합니다. 콜론 `<onnc_source_dir>` 앞의 경로는 다운로드한 ONNC 소스 코드의 절대 경로를 나타냅니다. 콜론 `/onnc/onnc` 뒤의 경로는 Docker 컨테이너의 절대 경로를 나타냅니다.

옵션 `-ti`는 Docker 컨테이너에서 Ubuntu 셸 프롬프트로 들어가는 대화형 모드를 활성화하는 것을 의미합니다.

마지막 인수 'onnc/onnc-community'는 ONNC Docker 이미지의 이름입니다. 위의 명령을 입력하면 다음과 같은 Docker 프롬프트가 표시됩니다.

```console
$ docker run -ti --rm -v ~/work/onnc_projects/onnc:/onnc/onnc onnc/onnc-community
onnc@1fda72903b5c:/onnc/onnc-umbrella/build-normal$ 
```
 
Docker 프롬프트 내에서 다음 명령을 입력하여 Foo라는 새 백엔드를 만듭니다.

```console
// Within the Docker prompt, go to the path where the source codes are mounted to.
$ cd /onnc/onnc

// Run the script to create a new backend called Foo.
$ ./scripts/create-new-backend.sh Foo
```

새로운 백엔드인 Foo의 소스 코드는 `/onnc/onnc/lib/Target/Foo` 폴더에 나타납니다. 실제로 해당 폴더는 템플릿 폴더 `/onnc/onnc/lib/Target/Vanilla`에서 복사되고 이에 따라 새 백엔드 이름으로 조정됩니다.

Docker 프롬프트를 종료하려면 다음 명령을 사용하십시오.

```console
$ exit
```

### 2.4. Compiling the new backend

새 백엔드가 생성되면 ONNC를 다시 컴파일하여 새 백엔드를 활성화해야 합니다. 다음 명령을 사용하여 ONNC와 새 백엔드를 컴파일하십시오.

```console
// Do the following within the Docker prompt.
$ cd /onnc/onnc-umbrella/build-normal/

// Use “-j8” to invoke 8 CPU cores to do the parallel compilation.
$ smake -j8 install
```

이러한 명령을 성공적으로 실행하면 다음과 같은 결과가 표시됩니다.

```console
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/Rope.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/HashTable.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/StringSwitch.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/NodeIterator.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/Flags.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/IListNode.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/StringHasher.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/StringHashTable.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/TypeSwitch.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/ADT/If.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Notation.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Reader.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Array.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Type.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Group.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Storage.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Value.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/String.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/JSON/Object.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/Config
-- Installing: /onnc/onnc-umbrella/install-normal/include/onnc/Config/Config.h
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/Config/ONNX.h
-- Installing: /onnc/onnc-umbrella/install-normal/include/onnc/Config/Backends.def
-- Installing: /onnc/onnc-umbrella/install-normal/include/onnc/Config/Platforms.def
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/Support
-- Up-to-date: /onnc/onnc-umbrella/install-normal/include/onnc/Support/DataTypes.h
onnc@705b08cf8a7d:/onnc/onnc-umbrella/build-normal$
```

### 2.5. Invoking the new backend

The following commands demonstrate processing the AlexNet model with the new backend. 

```console
// Do the following within the Docker prompt.
$ cd /onnc/onnc-umbrella/build-normal/

// Invoke the new backend Foo.
$ ./tools/onnc/onnc /models/bvlc_alexnet/model.onnx -mquadruple foo
```

Option `-mquadruple foo`<sup id="quadruple-1">[1](#quadruple)</sup> is for invoking the new backend Foo. Use all lowercase letters as the backend name in this option. 

The following console output shows the execution result of the new backend. It prints information of the operators in the AlexNet model. 

```console
onnc@705b08cf8a7d:/onnc/onnc-umbrella/build-normal$ ./tools/onnc/onnc /models/bvlc_alexnet/model.onnx -mquadruple foo
Foo is invoked
%conv1_w_0<float>[96, 3, 11, 11] = Initializer<unimplemented>()
%conv1_b_0<float>[96] = Initializer<unimplemented>()
%conv2_w_0<float>[256, 48, 5, 5] = Initializer<unimplemented>()
%conv2_b_0<float>[256] = Initializer<unimplemented>()
%conv3_w_0<float>[384, 256, 3, 3] = Initializer<unimplemented>()
%conv3_b_0<float>[384] = Initializer<unimplemented>()
%conv4_w_0<float>[384, 192, 3, 3] = Initializer<unimplemented>()
%conv4_b_0<float>[384] = Initializer<unimplemented>()
%conv5_w_0<float>[256, 192, 3, 3] = Initializer<unimplemented>()
%conv5_b_0<float>[256] = Initializer<unimplemented>()
%fc6_w_0<float>[4096, 9216] = Initializer<unimplemented>()
%fc6_b_0<float>[4096] = Initializer<unimplemented>()
%fc7_w_0<float>[4096, 4096] = Initializer<unimplemented>()
%fc7_b_0<float>[4096] = Initializer<unimplemented>()
%fc8_w_0<float>[1000, 4096] = Initializer<unimplemented>()
%fc8_b_0<float>[1000] = Initializer<unimplemented>()
%OC2_DUMMY_1<int64>[2] = Initializer<unimplemented>()
%data_0<float>[1, 3, 224, 224] = InputOperator<unimplemented>()
%conv1_1<float>[1, 96, 54, 54] = Conv<auto_pad: "NOTSET", dilations: [1, 1], group: 1, kernel_shape: [11, 11], pads: [0, 0, 0, 0], strides: [4, 4]>(%data_0<float>[1, 3, 224, 224], %conv1_w_0<float>[96, 3, 11, 11], %conv1_b_0<float>[96])
%conv2_1<float>[1, 256, 26, 26] = Conv<auto_pad: "NOTSET", dilations: [1, 1], group: 2, kernel_shape: [5, 5], pads: [2, 2, 2, 2], strides: [1, 1]>(%pool1_1<float>[1, 96, 26, 26], %conv2_w_0<float>[256, 48, 5, 5], %conv2_b_0<float>[256])
%conv3_1<float>[1, 384, 12, 12] = Conv<auto_pad: "NOTSET", dilations: [1, 1], group: 1, kernel_shape: [3, 3], pads: [1, 1, 1, 1], strides: [1, 1]>(%pool2_1<float>[1, 256, 12, 12], %conv3_w_0<float>[384, 256, 3, 3], %conv3_b_0<float>[384])
%conv4_1<float>[1, 384, 12, 12] = Conv<auto_pad: "NOTSET", dilations: [1, 1], group: 2, kernel_shape: [3, 3], pads: [1, 1, 1, 1], strides: [1, 1]>(%conv3_2<float>[1, 384, 12, 12], %conv4_w_0<float>[384, 192, 3, 3], %conv4_b_0<float>[384])
%conv5_1<float>[1, 256, 12, 12] = Conv<auto_pad: "NOTSET", dilations: [1, 1], group: 2, kernel_shape: [3, 3], pads: [1, 1, 1, 1], strides: [1, 1]>(%conv4_2<float>[1, 384, 12, 12], %conv5_w_0<float>[256, 192, 3, 3], %conv5_b_0<float>[256])
 = OutputOperator<unimplemented>(%prob_1<float>[1, 1000])
onnc@705b08cf8a7d:/onnc/onnc-umbrella/build-normal$
```

## 3. Files of the new backend

By following the commands in the previous section, we derive a new backend Foo and all the files are created under `/onnc/onnc/lib/Target/Foo`. Table 1 describes the files in the created folder. 

**Table 1: Files of a backend and their purposes.**

| File | Purpose |
| ---- | ------- |
| `FooBackend.cpp & .h` | The main file of a backend. Developers need to modify this file to add optimization passes. |
| `CodeEmitVisitor.cpp & .h` | Implementation of the `CodeEmitVisitor` class. Developers need to modify this file to handle the code generation for each operator. |
| `TargetInfo/FooTargetInfo.cpp & .h` | This file containing functions for registering this backend to the ONNC framework. |
| `TargetInfo/FooTargetMemInfo.cpp & .h` | The file for configuring  memory size and alignment for each data type in neural network models. Developers need to modify this file based on the target hardware attributes to optimize memory allocation. |
| `CMakeLists.txt` | Configuration file for the CMake building system. |
| `Makefile.am` | Configuration file for the Autotools building system. |


## 4. Customizing the new backend


### 4.1. Extending ONNC IR for Unsupported Operators

To support the target-specific operators that does not exist in the default ONNC IR, please refer to the application note, [ONNC IR Extension Guide](ONNC-IR-Extension-Guide.md), for more details.

### 4.2. Handling code generation in the `codeEmit` pass
The `codeEmit` pass is added to handle code generation for the target backend. Please refer to [The Code Emitting Pass User Guide](The-Code-Emitting-Pass-User-Guide.md) for more details.

### 4.3. Adding optimization passes

The generated backend has included some optimization algorithms by default. Each algorithm is implemented in ONNC as a “pass” (the same concept as the LLVM pass). The default optimization passes may not fit your need so you need to develop your own passes and then edit `FooBackend.cpp` to add those passes into the compilation flow. Below is an example of adding a pass. Refer to the application note, [Pass Manager Getting Started Guide](ONNC-Pass-Manager-Getting-Started-Guide.md),  for more details about how to add a pass.

```cpp
void FooBackend::addOnncIrOptimization(PassManager& pPM, OptimizationOptions& options)
{
  TargetBackend::addOnncIrOptimization(pPM, options);
  // One example of adding your optimization pass. 
  pPM.add<YourProprietaryPass>();
}
```
**Code Snippet 2. Example of adding an optimization pass into a backend.**
 
In the above example, the optimization pass is added in the method, `addOnncIrOptimization()`. There are five stages in the compilation flow for users to add passes. Each stage in the compilation flow is implemented in a corresponding method. The following table shows the meaning and input/output of each method. 

**Table 2. The five methods representing the five compilation phases.**

| Method | Input | Output | Description |
| ------ | ----- | ------ | ----------- |
| `addTensorSel` | **ONNX IR** | **ONNC IR** | This method contains passes for translating models in the ONNX format into ONNC IR. |
| `addOnncIrOptimization` | **ONNC IR** | **ONNC IR** in optimized order | This method contains passes for optimizing ONNC IR in order to better performance. |
| `addTensorSched` | **ONNC IR** in optimized order | **ONNC IR** in optimized order | This method contains passes for better scheduling the execution order of ONNC IR. |
| `addMemAlloc` | **ONNC IR** in optimized order | **ONNC IR** with addresses | This method contains passes for allocating memory space of input data, weights, and activation data. |
| `addCodeEmit` | **ONNC IR** with address | **Machine codes** | This method contains passes for handling code generation and optimization. |

ONNC provides a couple of basic optimization passes by default. They are listed in the following table. 

**Table 3. The default passes in each optimization phase.**

| Method | Default passes | Description |
| ------ | -------------- | ----------- |
| `addTensorSel` | `addStandardTensorSel` | This pass translates models in the ONNX format into ONNC IR. |
| `addTensorSched` | N/A | |
| `addMemAlloc` | `addStandardCreateLiveIntervals` | This pass calculates the liveness intervals of tensors (input/output of operators). |
| `addMemAlloc` | `addStandardMemoryAllocation` | This pass allocates addresses for tensors with the consideration of tensors’ liveness intervals. |
| `addMemAlloc` | `addStandardSetMemOperands` | This pass saves the result of memory allocation to an internal data structure called `MemoryOperand` so that the result can be accessed by other passes. |
| `addCodeEmit` | `CodeEmit` | This pass generates the target machine codes. Note that this pass initially is just empty, which needs backend developers to add target-dependent implementation. |

### 4.4. Rebuilding ONNC 
After all modification is done in the new backend, remember to rebuild ONNC by following the steps in Section 2.4. 


----

<b id="quadruple">[1]</b>: `Quadruple` in ONNC is a string used for encapsulating backend information. The string can contain rich target information such as version, vendor, operating system, toolchain, and etc. A funny example is `-mquadruple foo-apple-darwin-gnu-ar-0.9.3-onnc-ca7`. Here we set our quadruple string to contain only one information: a special name `foo` to represent the target backend. This is also the minimum requirement of such a string. [↩](#quadruple-1)
