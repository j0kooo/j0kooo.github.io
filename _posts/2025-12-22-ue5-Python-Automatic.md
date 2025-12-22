---
layout: post
title: "[UE5] Python 자동화"
description: 언리얼 에디터 반복 작업, Python으로 끝내기
date: 2025-12-22 09:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Python, dd]
---
## 개요

게임 개발을 하다보면, 반복적인 작업이 필요하고는 합니다. 작업의 반복 횟수가 적다면 상관없지만, 100개 단위만 넘어가도 “아 이걸 언제 다하지?”라는 생각이 들고는 합니다. 이때 언리얼 엔진은 이런 반복 작업을 자동화할 수 있도록 Python을 지원합니다. 

이번 포스트에서는 언리얼에서 Python을 사용할 때의 "<u>이점, 환경 구성 방법, 실제 적용 예시</u>" 까지 정리해보겠습니다.

<br>

## 왜 언리얼에서 Python을 써야 하는가?

위에서 설명했듯 저는 보통 반복 작업에 많이 사용하고는 합니다. 예를 들어서 수백 개의 텍스처의 이름을 바꾸거나, 수백 개의 에셋을 만들어야 할 때와 같이 말이죠. 아니면 기획자가 작성한 csv 파일을 StringTable에 적용시키는 자동화를 위해서도 사용했습니다.

그렇다면 언리얼에서 사용하는 C++를 사용하지 않고, 하필 Python을 사용할까요? 제가 생각하는 가장 큰  장점은 빌드 여부입니다. C++는 수정 시 컴파일 과정을 거쳐야 하지만, Python은 에디터 런타임에서 즉시 실행할 수 있기 때문에 툴과 자동화 영역에서 반복 작업에 훨씬 유리합니다.

<br>

![왜 파이썬인가](/assets/img/post/Python/WhyPython.png){: width="80%" style="display:block; margin: 0 auto;"}
*왜 파이썬인가*

이건 저의 생각이였고, 언리얼 공식 문서를 보면, Python은 툴 간 호환성과 파이프라인 자동화를 위해서 업계 표준으로 채택되었으며, 초보자가 사용하기에 상대적으로 쉬운 이점이 있다고 하네요.

<br>

## 언리얼 Python 환경 구성하기

일단 IDE가 필요할텐데, 저는 VSCode를 기준으로 설명하겠습니다.

### IDE 설정

1. VSCode 설치
2. [Unreal Engine Python](https://marketplace.visualstudio.com/items?itemName=NilsSoderman.ue-python) 설치

    ![Unreal Python IDE](/assets/img/post/Python/IDEUnrealEnginePython.png){: width="80%" style="display:block; margin: 0 auto;"}
    *Unreal Python IDE*

    먼저 "IDE > Plugin" 탭에서 위 플러인을 찾아 설치 및 활성화해줍니다.

3. [자동 완성 세팅](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/setting-up-autocomplete-for-unreal-editor-python-scripting)

    <div style="display: flex; justify-content: center; gap: 10px;">
    <figure style="text-align: center; width: 50%;">
        <img
        src="/assets/img/post/Python/Setting1.png"
        alt="설정 1"
        style="width: 80%; border-radius: 8px;"
        >
    </figure>
    <figure style="text-align: center; width: 50%;">
        <img
        src="/assets/img/post/Python/Setting2.png"
        alt="설정 2"
        style="width: 80%; border-radius: 8px;"
        >
    </figure>
    </div>

    자동 완성을 위한 설정을 해주는데요. 위 링크에 들어가서 참고 하시면 편합니다.
    
4. (선택) PC에 Python 설치

    추가로 에디터를 키지 않고도 에셋을 만들고 싶은 경우에는 PC에 맞는 Python 버전을 설치해야만 합니다. 언리얼 엔진 플러그인에 Python이 포함되어 있기 때문에 에디터 내부에서 실행할 거면 별도 Python 설치는 없어도 됩니다.

<br>

### Unreal 설정

1. Python 플러그인 설치

    ![Python Editor Script Plugin](/assets/img/post/Python/PythonEditorScriptPlugin.png){: width="80%" style="display:block; margin: 0 auto;"}
    *Python Editor Script Plugin* 

    에디터에서 Plugin 탭에 들어가 "Python Editor Script Plugin"을 찾아 활성화해줍니다.
    
2. 환경 설정
    
    ![Plugin Setting](/assets/img/post/Python/PluginSetting.png){: width="80%" style="display:block; margin: 0 auto;"}
    *Plugin Setting* 

    py 파일들이 있을 폴더를 설정하여 외부에서 접근할 수 있도록 지정해줍니다.

<br>

## 실행 방법

1. .py 스크립트 작성
2. "Ctrl + Shift + P"로 "Unreal Python" 검색

    ![Attach And Execute](/assets/img/post/Python/AttachAndExecute.png){: width="80%" style="display:block; margin: 0 auto;"}
    *Attach And Execute* 
    
    위 그림과 같이 "Unreal Python: Attach"와 "Unreal Python: Excute"가 있는데, 아래와 같아요.

    - Unreal Python: Attach → 현재 켜져있는 에디터에 바인딩해서 py 파일에 대해 디버깅을 할 수 있도록 함
    - Unreal Python: Excute → 실행 (바인딩 되어 있지 않아도 가능)

<br>

## 실제 사례

아래는 제가 실제로 해봤던 자동화들입니다. 코드는 공유가 어렵지만, "이런 것도 가능하다" 정도로만 알아도 좋을 것 같습니다.

<br>

### 1. BP 특정 컴포넌트의 Mesh 지정

Blueprint 내부에 StaticMesh 컴포넌트에 대해서 새로 추가된 리소스를 적용해야 했는데요. 100개가 넘는 양이였기 때문에 여기에 `Python`을 사용해봤습니다. BP 이름에 규칙이 있었고, 새로 적용할 StaticMesh 이름에도 규칙이 있었기 때문에 일괄 처리가 가능했죠. 과정은 아래와 같습니다.

1. 타겟 Blueprint가 있는 폴더에서 Blueprint에 접근
2. Blueprint 내부에 있는 특정 이름의 컴포너트를 찾아
3. 이름 규칙에 맞는 StaticMesh를 지정

<br>

---

### 2. 시퀀스 제작

이것도 위 예시와 비슷한데요. Sequence을 지정한 이름, 폴더 규칙에 맞춰 **대량 생성**했습니다. 사람이 하면 오타나 경로 실수가 필연적으로 섞이는데, 스크립트로 만들면 그 리스크가 거의 사라지는 장점이 있죠. 

1. 네이밍 규칙에 맞게 조합하여 이름을 지정
2. 지정된 경로에 uasset을 생성
    
<br>

---

### 3. Localization 자동화

이건 개인적으로 가장 임팩트가 컸습니다. 위 예시들과는 다르게 1회성이 아닌 빈도 높은 작업이기도 하고, 아직까지도 잘 사용하고 있거든요. 진행 흐름은 아래와 같습니다.

1. CSV 파일을 분석해서 각 언어에 대한 확장자 파일인 `.archive` 생성
    - 추가로 기본 언어 StringTable 생성
2. Localization Dashboard에서 Gather와 Compile 실행
3. 결과로 `.locres` 생성

즉, **CSV 변경 → 로컬라이징 리소스 갱신**까지의 파이프라인을 자동화하여 소요 시간을 단축할 수 있었죠.

<br>

> 이외에도 Static Mesh의 레벨 오브 디테일(LOD) 생성과 같은 [시간 소모가 많은 에셋 관리 작업을 자동화](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/creating-levels-of-detail-in-blueprints-and-python-in-unreal-engine)할 수 있다고 하네요.
{: .prompt-tip }

<br>

## 팁

### **1. init_unreal.py**

![init_unreal.py](/assets/img/post/Python/init_unreal.png){: width="80%" style="display:block; margin: 0 auto;"}
*init_unreal.py* 

에디터가 사용하는 Python 경로에서 `init_unreal.py` 를 발견하면 **즉시 자동 실행**합니다. 프로젝트/플러그인 작업에서 "누가 작업하든 동일한 초기화 코드가 필요"한 경우에 유용합니다. 이 이름으로 된 스크립트 안에 초기화 코드를 넣은 뒤 그 프로젝트 또는 플러그인 안의 "**Content > Python**" 폴더에 넣으면 됩니다.

---

### **2. 스타트업 스크립트**

![Startup Script](/assets/img/post/Python/StartupScript.png){: width="80%" style="display:block; margin: 0 auto;"}
*Startup Script* 

프로젝트 세팅에서 프로젝트를 열 때마다 실행할 Python 스크립트를 원하는 만큼 지정할 수 있습니다. 에디터는 기본 스타트업 레벨 로드가 끝나면 이러한 스크립트를 실행한다고 하네요. "**Edit > Project Settings > Plugins > Python" 에서** 스크립트를 **스타트업 스크립트(Startup scripts)** 세팅에 추가합니다.

<br>

> 물론 저는 위 2가지 모두 사용해본 적은 없습니다.
{: .prompt-tip }

<br>

## 마무리

`Python`을 사용하면, 반복 작업을 편하게 해주기도 하고, 대규모 작업에 대해서 휴먼 에러까지 잡을 수 있는 유용한 도구입니다. 나름의 재미도 있고 활용할 수 있는 범위도 넓으니, 반복 작업을 사람이 계속 하고 있다면, 한번 시도해 보는 것도 좋겠습니다.

<br>

> 물론 관련 자료가 별로 없다는 치명적인 단점이 있습니다... ㅎ
{: .prompt-warning }

<br>

### 참고 자료

- [[공식 문서] Python 자동 완성 세팅](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/setting-up-autocomplete-for-unreal-editor-python-scripting)
- [[공식 문서] Python을 사용한 언리얼 에디터 스크립팅](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/scripting-the-unreal-editor-using-python)