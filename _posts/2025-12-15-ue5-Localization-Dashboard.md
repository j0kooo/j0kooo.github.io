---
layout: post
title: "[UE5] 텍스트 현지화와 Localization Dashboard"
description: 언리얼 엔진에서의 현지화 방법과 Localization Dashboard
date: 2025-12-15 09:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Localization, Localization Dashboard]
---
## 개요
게임을 출시하기 전에는 생각보다 많은 것들을 고려해야 합니다. 서비스할 국가는 어디인지, 어떤 언어를 지원할 건지, 심지어는 특정 국가에서 사용하지 못하는 단어나 이미지까지도 고려하는 경우도 있습니다.

이처럼 현지화는 글로벌 출시를 위한 기본 단계입니다. 인디 게임을 출시 하려고 한다면 꼭 알아야 하죠. 그럼 언리얼 엔진에서는 현지화를 어떤 방식으로 지원할까요?

<br>

## 기존 현지화의 문제점

| Index | KR | EN |
| --- | --- | --- |
| 1 | 시작 | Start |
| 2 | 종료 | End |

저는 처음에 위와 같이 인덱스와 언어별로 번역된 문장이 저장된 테이블을 통해서 언어가 바뀌면 위 테이블에서 언어 컬럼에서 문자열을 가져오는 방식으로 진행했었습니다. 

구현 자체도 단순하고 언어 전환에 큰 문제는 없었지만, <u>언어 갱신시 발생하는 이벤트를 각 텍스트 블럭에 바인딩을 처리해야 되서 중복되는 코드가 증가하는 문제</u>가 있었습니다. 엔진에서 제공하는 현지화 시스템과는 다르기도 하고, <u>이미지나 에셋에 대해서는 현지화가 적용되지 않는다는 문제점</u>도 있었죠.

그래서 엔진에서 현지화를 지원하는 것이 무엇인가 찾다가 `Localization Dashboard` 에 대해서 알게되었습니다. 이는 설정된 언어에 따라서 텍스트뿐만 아니라 이미지와 같은 에셋이나 폰트에 대해서도 현지화가 지원합니다. 

이 글에서는 텍스트의 현지화를 중심으로 간단한 사용 방법과 장단점, 경험 중심으로 정리해보겠습니다.

<br>

## 현지화 과정
![텍스트 현지화 과정](/assets/img/post/LocalizationDashboard/LocalizationDashboardFlow.png){: width="580" height="295"}
*텍스트 현지화 과정*

언리얼 엔진의 텍스트 현지화 과정은 크게 다음과 같은 순서로 진행됩니다.

1. **Localization Dashboard를 통해 프로젝트 내의 패키지 파일이나 텍스트 리소스에서 현지화 대상 텍스트를 수집**
2. **지원할 언어를 추가한 뒤, 각 언어별로 번역본을 작성하거나 외부 번역 작업을 진행**
3. **번역 결과를 읽을 수 있도록 컴파일**

설명만 들었을 때는 모든 위젯이나 BP등 에셋에 존재하는 모든 텍스트를 자동으로 수집해주니 개발자는 현지화만 해주면 되는 아주 편리한 도구로 인식되지만, 조금만 더 생각해보면 유지보수 측면에서 뚜렷한 한계가 보입니다.

<br>

## 문제 - 왜 유지보수가 어려워질까?

`Localization Dashboard`가 모든 에셋을 탐색해 텍스트를 수집한다는 말은, 반대로 말하면 **텍스트가 프로젝트 곳곳에 흩어질 수밖에 없다는 뜻**이기도 합니다.

작은 프로젝트에서는 크게 체감이 안 되지만, 규모가 커질수록 텍스트의 출처를 일일이 파악하기가 어렵고, 특정 문구를 수정하거나 교체하려는 경우 전체 프로젝트를 스캔해야 합니다. 특히, 다수의 디자이너나 개발자가 협업하는 환경에서는 동일한 문구가 여러 위젯에 중복될 가능성이 높고, 이로 인해 일관성 유지가 어려워지겠죠.

<br>

### 해결 - 수집 범위를 통제하자

그래서 저는 모든 텍스트를 **한 곳에서 통합 관리**하는 방향을 선택했습니다. 특정 문구의 수정, 추가, 삭제가 필요할 때도 테이블만 수정하면 되고, 실수로 특정 텍스트가 누락되거나 중복 정의되는 문제를 방지할 수 있죠.

즉, 해당 테이블에 대해서만 `Localization Dashboard`에서 수집하게 하고, 현지화 과정을 거친 후에 해당 텍스트를 위젯이나 필요한 곳에 바인딩하는 방식입니다.  이렇게 하면 유지보수성이 크게 향상될 뿐만 아니라, 프로젝트 규모가 확장되더라도 현지화 작업의 안정성을 보장할 수 있습니다.

<br>

## 수집 방법

### 1. Localization Dashboard을 열고, 타겟을 지정

![타겟 지정](/assets/img/post/LocalizationDashboard/SetTarget.png){: width="717" height="298"}
*타겟 지정*
        
Game Targets을 추가해서 이름을 지정합니다. 이때 GameTargets 이름은 단순한 구분자의 역할이고, 보통은 '에디터, 빌드, 플랫폼'을 구분하기 위한 용도로 사용합니다. 

---

### 2. 텍스트를 수집할 파일 경로 지정
    
![수집할 텍스트 경로 지정](/assets/img/post/LocalizationDashboard/SetGatherPath.png){: width="602" height="328"}
*수집할 텍스트 경로 지정*
        
'Uasset, Umap'와 같은 패키지 파일이나 'h, cpp, ini'와 같은 텍스트 파일의 경로를 지정할 수 있습니다. `StringTable`에 대한 접근만 할 것이라면, Gather from Packages에서 지정해주면 됩니다.

---

### 3. Loading Policy를 타입에 맞게 지정
    
1번에서 설명했듯 **Localization 리소스가 게임 실행 시 언제, 어떻게 로드될지를 결정하는 정책입니다.** 즉 번역 데이터가 언제 메모리에 올라오는지를 제어하는 옵션이라고 생각하면 됩니다.
    
| 옵션 | 의미 | 실무 기준 조언 |
| --- | --- | --- |
| **Never** | 해당 데이터는 아예 로드되지 않음 | 레거시의 경우 사용 |
| **Editor** | 에디터에서만 로드, 런타임에서는 무시 | 에디터 편집용 더미 데이터에 사용 가능 |
| **Game** | 게임 실행 시 해당 언어 데이터 항상 로드 | 일반적인 번역 적용은 대부분 Game 설정  |
| **Always** | 에디터 & 게임 모두 무조건 로드 | 핵심 언어 데이터, 항상 로드가 필요한 경우 추천 |

---

### 4. 지원 언어를 추가하고 Default 지정
    
![디폴트 언어 지정](/assets/img/post/LocalizationDashboard/SetDefault.png){: width="703" height="143"}
*디폴트 언어 지정*
    
한글과 영어를 추가했고, 한글을 기본 언어로 설정했습니다.

---

### 5. Gather Text를 통해서 디폴트 언어에 대한 텍스트들을 가져옴
    
![텍스트 수집](/assets/img/post/LocalizationDashboard/GatherText.png){: width="585" height="321"}
*텍스트 수집*

---

### 6. 각 언어에 대한 현지화를 진행
    
![현지화 번역 진행](/assets/img/post/LocalizationDashboard/Localization.png){: width="702" height="359"}
*현지화 번역 진행*
    
각 언어에 대해서 번역을 작성 후 저장하고, 컴파일을 눌러줍니다.

---

### 7. 관련 리소스가 생성된 것을 확인
    
![메타 데이터](/assets/img/post/LocalizationDashboard/metadata.png){: width="536" height="186"}
*메타 데이터*
    
'Content > Localization' 폴더에 가보면, 관련된 파일들이 생성된 것을 볼 수 있습니다.

<br>

> 아마 위처럼 진행했다면, 'Editor Preferences > Region & Language > Preivew Game language'에 언어들이 추가 된 것을 볼 수 있는데, 이게 잘 적용이 된 상태입니다.
{: .prompt-tip }

<br>

## 텍스트 지정 방법

### 1. 정적 지정

![정적 텍스트 지정](/assets/img/post/LocalizationDashboard/SetStatic.png){: width="683" height="202"}
*정적 텍스트 지정*

텍스트 옆의 깃발 아이콘을 통해서 `String Table` 에 접근하고, 원하는 텍스트를 찾아 지정합니다. `Editor Preferences > Region & Language > Preview Game Language`를 바꿔가며 미리보기로 확인할 수 있습니다.

---

### 2. 동적 지정 (BP)

![동적 지정](/assets/img/post/LocalizationDashboard/SetDynamic.png){: width="1228" height="186"}
*동적 지정*

포럼에서는 좌측 그림과 같이 보통 `Make Literal Text` 노드만을 사용해서 “정적 지정”하는 방식으로 하는 예시가 많습니다. 다만 실무에서는 Key를 런타임에서 바꿔야 하는 경우가 많아서, **StringTable을 직접 참조해서 Key를 동적으로 구성**하는 쪽으로 구현합니다.

---

### 3. 동적 지정 (C++)

```cpp
FText ULocalizationSubsystem::GetLocalizedTextFromTable(const int32 Index)
{
	FText Text = FText::FromStringTable("/Game/Localization/Game/ST_Localization.ST_Localization", *FString::FromInt(Index));
	return Text;
}
```

BP에서 사용하는 `MakeTextFromStringTable` 의 동작 방식과 동일하게 `StringTable`의 경로만으로 접근해서 `FText`를 반환하는 방식입니다. 이는 범용적으로 사용할 것 이기 때문에 `Subsystem`으로 빼서 사용하는 것을 추천합니다.

<br>

## 언어 전환 방법

![언어 전환](/assets/img/post/LocalizationDashboard/SwitchLocalization.png){: width="524" height="180"}
*언어 전환*

그렇다면 런타임 중에 설정을 통해서 언어를 바꾸는 경우 어떻게 대응해야할까요? 런타임 중에는 `GetCurrentCulture`를 통해서 현재 국가의 코드를 가져오고, `SetCurrentCulture`를 통해서 국가 코드를 넘겨 언어를 지정할 수 있습니다.

이때 'Culture, Language, Local'이 있는데, Language는 언어, Locale은 지역, Culture는 이 2가지를 모두 한번에 포함한 정보를 의미합니다.

> 이때 국가 코드는 IETF 표준 코드 규격을 따라가는데, ISO 639인 언어코드와 ISO 3166인 국가/지역 코드를 합쳐서 사용합니다.
{: .prompt-tip }

<br>

### 문제 - 왜 에디터가 바뀔까

`SetCurrentCulture`를 사용하면, <u>PIE(Viewport)에서 실행하면 게임 텍스트가 바뀌지 않고, 에디터 언어만 바뀐 것처럼 보이는 상황</u>이 나올 수 있습니다.

```cpp
bool UKismetInternationalizationLibrary::SetCurrentCulture(const FString& Culture, const bool SaveToConfig)
{
	if (FInternationalization::Get().SetCurrentCulture(Culture))
	{
		if (!GIsEditor && SaveToConfig)
		{
			GConfig->SetString(TEXT("Internationalization"), TEXT("Culture"), *Culture, GGameUserSettingsIni);
			GConfig->EmptySection(TEXT("Internationalization.AssetGroupCultures"), GGameUserSettingsIni);
			GConfig->Flush(false, GGameUserSettingsIni);
		}
		return true;
	}

	return false;
}
```

코드상으로도 에디터에서는 ini를 저장하지 않기 때문에, 플레이 환경에 따라 다르게 동작하는 문제가 생기는 것입니다. 

<br>

### 해결 - 상태에 따른 분기

```cpp
void ULocalizationSubsystem::RefreshLanguageResources(const FString& CultureName)
{
	FTextLocalizationManager::Get().RefreshResources();

#if WITH_EDITOR
	if (GIsEditor)
	{
		FTextLocalizationManager::Get().EnableGameLocalizationPreview(CultureName);
	}
#else
	UKismetInternationalizationLibrary::SetCurrentCulture(CultureName, true);
#endif
}
```

그래서 이에 대한 대응법으로 위 코드를 작성해봤는데요. 에디터에서는 Preview를 명시적으로 켜주고, 그 외 빌드에서는 일반적인 `SetCurrentCulture` 흐름을 타게 분기하는 방식입니다.

<br>

## 그 외 팁

### 1. PO를 이용한 번역

에디터에서 수집된 텍스트에 대해서 하나씩 번역해도 되지만, 수집된 데이터를 PO파일로 내보내고, 이를  [Poedit](https://poedit.net/)으로 수정할 수 있습니다. PO로 내보내고, 외부 업체에 맡겨서 번역한 다음에 다시 PO를 임포트하는 것도 하나의 방법입니다.

> 이외에도 만약 csv로 현지화 테이블을 관리하는 경우에는 Python으로 자동화해서 수집하는 방법도 있습니다. 저는 이 방법을 사용했습니다.
{: .prompt-tip }

---

### 2. 패키징 시 주의사항

![패키지 설정](/assets/img/post/LocalizationDashboard/PackagingSetting.png){: width="507" height="162"}
*패키지 설정*

빌드에서 번역이 안 보이는 경우, 먼저 `Localizations to Package`에 해당 언어가 포함돼 있는지 확인하는 게 우선입니다.

---

### 3. String Table 키 전략

저는 `String Table` 엔트리 키를 숫자 인덱스로 운영했습니다. 수정이 잦지 않은 문구에 대해서는 키가 잘 변하지 않아서 관리가 편하다는 장점이 있습니다. 물론 인덱스로만 하면 구별과 추적이 어렵다는 문제가 있습니다. 

<br>

## 마무리

`Localization Dashboard`는 언리얼에서 제공하는 훌륭한 현지화 파이프라인입니다. 엔진에서 언어 현지화를 관리해주니 알아두면 좋을 것 같고, 저도 에셋과 관련해서 현지화 작업은 안해봤지만 나중에 사용해보고 또 올려보겠습니다.

하지만 앞서 말했듯이 프로젝트가 커질수록 **"자동 수집"**은 오히려 운영 리스크가 될 수 있으니, 하나의 테이블에서 관리하는 방향을 추천드립니다!

<br>

## 출처 및 참고자료

- [Localization In-Depth](https://dev.epicgames.com/community/learning/tutorials/zwPJ/unreal-engine-localization-in-depth)
- [Localization feature of ue4](https://www.docswell.com/s/EpicGamesJapan/ZPMRQZ-UE_LocalizationDD_Features_En#p31)
- [Localized Strings Using StringTable and C++](https://unreal-garden.com/tutorials/stringtable-cpp/#2-modify-our-game-module-class)
- [Docs String Table](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-string-tables-for-text-in-unreal-engine)
- [Unreal UIs and Localization](https://unreal-garden.com/tutorials/ui-localization/)