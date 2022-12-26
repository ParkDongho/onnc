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

최신 ONNC 소스 코드는 GitHub에서 사용할 수 있습니다. 소스 코드를 다운로드하려면 다음 명령을 따르십시오.

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

다음 명령은 새 백엔드로 AlexNet 모델을 처리하는 방법을 보여줍니다.

```console
// Do the following within the Docker prompt.
$ cd /onnc/onnc-umbrella/build-normal/

// Invoke the new backend Foo.
$ ./tools/onnc/onnc /models/bvlc_alexnet/model.onnx -mquadruple foo
```

`-mquadruple foo`<sup id="quadruple-1">[1](#quadruple)</sup> 옵션은 새로운 백엔드 Foo를 호출하기 위한 것입니다. 이 옵션에서 백엔드 이름으로 모두 소문자를 사용하십시오.

다음 콘솔 출력은 새 백엔드의 실행 결과를 보여줍니다. AlexNet 모델의 연산자 정보를 출력합니다.

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

이전 섹션의 명령을 따라 새로운 백엔드 Foo를 파생시키고 모든 파일은 `/onnc/onnc/lib/Target/Foo` 아래에 생성됩니다. 표 1은 생성된 폴더의 파일을 설명합니다.

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

기본 ONNC IR에 존재하지 않는 대상별 연산자 지원에 대한 자세한 내용은 [ONNC IR 확장 가이드](ONNC-IR-Extension-Guide.md) 애플리케이션 노트를 참조하십시오.

### 4.2. Handling code generation in the `codeEmit` pass
대상 백엔드에 대한 코드 생성을 처리하기 위해 `codeEmit` 패스가 추가되었습니다. 자세한 사항은 [The Code Emitting Pass User Guide](The-Code-Emitting-Pass-User-Guide.md)를 참고하시기 바랍니다.

### 4.3. Adding optimization passes

생성된 백엔드는 기본적으로 일부 최적화 알고리즘을 포함하고 있습니다. 각 알고리즘은 ONNC에서 "통과"(LLVM 통과와 동일한 개념)로 구현됩니다. 기본 최적화 패스는 필요에 맞지 않을 수 있으므로 고유한 패스를 개발한 다음 `FooBackend.cpp`를 편집하여 해당 패스를 컴파일 흐름에 추가해야 합니다. 다음은 패스를 추가하는 예입니다. 자세한 패스 추가 방법은 애플리케이션 노트 [패스 매니저 시작하기 가이드](ONNC-Pass-Manager-Getting-Started-Guide.md)를 참조하세요.

```cpp
void FooBackend::addOnncIrOptimization(PassManager& pPM, OptimizationOptions& options)
{
  TargetBackend::addOnncIrOptimization(pPM, options);
  // One example of adding your optimization pass. 
  pPM.add<YourProprietaryPass>();
}
```
**Code Snippet 2. Example of adding an optimization pass into a backend.**
 
위의 예에서 `addOnncIrOptimization()` 메서드에 최적화 패스가 추가되었습니다. 사용자가 패스를 추가할 수 있는 컴파일 흐름에는 5단계가 있습니다. 컴파일 흐름의 각 단계는 해당 메서드에서 구현됩니다. 다음 표는 각 메소드의 의미와 입/출력을 보여줍니다.

**Table 2. The five methods representing the five compilation phases.**

| Method | Input | Output | Description |
| ------ | ----- | ------ | ----------- |
| `addTensorSel` | **ONNX IR** | **ONNC IR** | 이 메서드에는 ONNX 형식의 모델을 ONNC IR로 변환하기 위한 패스가 포함되어 있습니다. |
| `addOnncIrOptimization` | **ONNC IR** | **ONNC IR** in optimized order | 이 메서드에는 성능 향상을 위해 ONNC IR을 최적화하기 위한 패스가 포함되어 있습니다. |
| `addTensorSched` | **ONNC IR** in optimized order | **ONNC IR** in optimized order | 이 메서드에는 ONNC IR의 실행 순서를 더 잘 예약하기 위한 패스가 포함되어 있습니다. |
| `addMemAlloc` | **ONNC IR** in optimized order | **ONNC IR** with addresses | 이 메서드에는 입력 데이터, 가중치 및 활성화 데이터의 메모리 공간을 할당하기 위한 패스가 포함되어 있습니다. |
| `addCodeEmit` | **ONNC IR** with address | **Machine codes** | 이 메서드에는 코드 생성 및 최적화를 처리하기 위한 패스가 포함되어 있습니다. |

ONNC는 기본적으로 몇 가지 기본 최적화 단계를 제공합니다. 다음 표에 나열되어 있습니다.

**Table 3. The default passes in each optimization phase.**

| Method | Default passes | Description |
| ------ | -------------- | ----------- |
| `addTensorSel` | `addStandardTensorSel` | 이 패스는 ONNX 형식의 모델을 ONNC IR로 변환합니다. |
| `addTensorSched` | N/A | |
| `addMemAlloc` | `addStandardCreateLiveIntervals` | 이 패스는 텐서(연산자의 입력/출력)의 활성 간격을 계산합니다. |
| `addMemAlloc` | `addStandardMemoryAllocation` | 이 패스는 텐서의 활성 간격을 고려하여 텐서에 대한 주소를 할당합니다. |
| `addMemAlloc` | `addStandardSetMemOperands` | 이 패스는 메모리 할당 결과를 'MemoryOperand'라는 내부 데이터 구조에 저장하여 다른 패스에서 결과에 액세스할 수 있도록 합니다. |
| `addCodeEmit` | `CodeEmit` | 이 패스는 대상 머신 코드를 생성합니다. 이 패스는 처음에는 비어 있으며 백엔드 개발자가 대상 종속 구현을 추가해야 합니다. |

### 4.4. Rebuilding ONNC 
새 백엔드에서 모든 수정이 완료되면 섹션 2.4의 단계에 따라 ONNC를 다시 빌드해야 합니다.

----

<b id="quadruple">[1]</b>: ONNC의 'Quadruple'은 백엔드 정보를 캡슐화하는 데 사용되는 문자열입니다. 문자열에는 버전, 공급업체, 운영 체제, 도구 체인 등과 같은 풍부한 대상 정보가 포함될 수 있습니다. 재미있는 예는 `-mquadruple foo-apple-darwin-gnu-ar-0.9.3-onnc-ca7`입니다. 여기서 우리는 하나의 정보만 포함하도록 4중 문자열을 설정합니다. 대상 백엔드를 나타내는 특수 이름 `foo`입니다. 이는 이러한 문자열의 최소 요구 사항이기도 합니다. [↩](#quadruple-1)
