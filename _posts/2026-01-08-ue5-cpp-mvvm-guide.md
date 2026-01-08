---
layout: post
title: "[UE5] C++로 구축하는 MVVM 패턴 실무 가이드"
description: C++ 기반 MVVM 시스템을 구축하는 예시 진행
date: 2026-01-08 09:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, MVVM, Model Veiw VeiwModel]
---
## 개요
지난 포스트에서 `MVVM`의 이론적인 배경을 다뤘다면, 이번에는 실전입니다. 언리얼 공식 문서를 뒤져보신 분들은 아시겠지만, 대부분의 예제가 블루프린트(BP) 위주로 되어 있습니다. 프로토타이핑에는 좋지만, 복잡한 로직과 데이터 관리, 그리고 형상 관리를 고려해야 하는 실무 환경에서는 C++ 기반의 구조가 절실해지는 시점이 반드시 옵니다.

이번 글에서는 C++로 MVVM 구조를 잡는 과정을 정리해 보았습니다.

> 기본적인 MVVM의 개념 설명은 생략합니다. 플러그인 설치 방법이나 기초적인 에디터 조작은 [공식 문서](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/umg-viewmodel-for-unreal-engine)를 참고해 주세요. 우리는 코드 레벨의 아키텍처에 집중합니다.
{: .prompt-info }

<br>

## ViewModel의 소유권 결정
`View`와 `Model`을 바인딩하기 전, 가장 먼저 선택해야하는 건 **"ViewModel의 생명주기를 누가, 어떻게 관리할 것인가?"**입니다. 이는 단순한 취향 차이가 아니라, <b><u>데이터의 성격과 수명</u></b>에 따라 결정되어야 합니다. 2가지로 구분해봤습니다.

1. **공유 인스턴스**
  - 게임 전체에서 싱글톤처럼 유지되어야 하는 상태
  - **Use Case** : 현재 점수, 플레이 시간과 같은 글로벌 설정
  - **LifeCycle** : 특정 위젯에 종속되지 않도록 관리

2. **개별 인스턴스**
  - 각 View가 독립적으로 가지는 상태
  - **Use Case** : 적의 체력, 인벤토리 슬롯과 같은 개별 데이터
  - **LifeCycle** : 위젯(View)이 생성될 때 만들어지고, 파괴될 때 함께 정리 (선택 사항)

<br>

### Creation Type 분석

<table style="margin-left:auto;margin-right:auto;">
  <thead>
    <tr>
      <th>Creation Type</th>
      <th>공유</th>
      <th>개별</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>Manual</td><td>O</td><td>O</td></tr>
    <tr><td>Create Instance</td><td>X</td><td>O</td></tr>
    <tr><td>Global View Model Collection</td><td>O</td><td>X</td></tr>
    <tr><td>Property Path</td><td>O</td><td>O</td></tr>
    <tr><td>Resolver</td><td>O</td><td>O</td></tr>
  </tbody>
</table>

위에서 소유권이나 형태를 지정했다면, 생성 타입을 지정해야 합니다. 언리얼에서 제공하는 `ViewModel`의 생성 타입은 위와 같이 5가지가 있는데, 나름대로 사용 방식에 따라 사용하기 좋은 타입을 구분해봤습니다. 

`CreateInstance` 는 위젯 생성과 동시에 ViewModel이 같이 생성되며, 바인딩되기 때문에 개별로 가지는 방식에 유용하고, `GlobalViewModelCollection` 은 말그대로 Global로 관리되기 때문에 공유하는 방식에 어울리겠습니다. `Manual`은 직접 바인딩을 해줘야되기 때문에 공유와 개별 모두 가능하지만, 개별에 조금 더 적극적으로 사용할 수 있을 것 같네요. 아래에서 3가지 방법 모두 설명해보겠습니다.

> `Property Path`와 `Resolver`는 외부에서 생성된 후에 `ViewModel`을 찾아 반환하는 형식을 따릅니다. 이건 다음 포스트에서 진행하겠습니다.
{: .prompt-tip }

<br>

## 실습 - 체력 시스템 구현 (To View)
먼저 가장 일반적인 상황인 `Model -> ViewModel -> View` 의 흐름을 구현하기 위한 작업을 진행해보겠습니다.

### 1. ViewModel 구성

`ViewModel` 인스턴스 타입에 따라 구분하기에 앞서 간단한 예시로 플레이어의 체력을 표현하기 위한 `ViewModel`을 구현해보겠습니다.

---

#### 1-1. 모듈 추가 및 클래스 생성
    
```cpp
PublicDependencyModuleNames.AddRange(new string[]{~, "ModelViewViewModel"});
```
    
가장 먼저 `{모듈 명}.Build.cs` 에 `ModelViewViewModel` 플러그인 모듈을 추가해서 의존성을 추가해줍니다.

<br>
    
```cpp
class PROJECTMVVM_API UStatusViewModel : public UMVVMViewModelBase
```
    
`UMVVMViewModelBase` 을 부모로 하는 자식 클래스를 하나 만들어줍니다. 보통 한 화면을 구성하도록 `ViewModel` 을 구성하거나 성격에 따라 구성하는게 유지보수에 유리합니다. 예를 들어 “인벤토리, 마우스, 스킬” 이런 식으로 나눠서 구성할 수 있겠습니다. 

<br>

---

#### 1-2. ViewModel 클래스 구성
    
```cpp
/** Header */
public:
	UFUNCTION()
	void SetCurrentHealth(const float InValue);
	float GetCurrentHealth() const { return CurrentHealth; }
	
	UFUNCTION()
	void SetMaxHealth(const float InValue);
	float GetMaxHealth() const { return MaxHealth; }

	UFUNCTION(BlueprintPure, FieldNotify)	
	float GetHealthRatio() const;

private:
	UPROPERTY(FieldNotify, Setter, Getter)
	float CurrentHealth;
	
	UPROPERTY(FieldNotify, Setter, Getter)
	float MaxHealth;

/** CPP */
void UStatusViewModel::SetCurrentHealth(const float InValue)
{
	if(UE_MVVM_SET_PROPERTY_VALUE(CurrentHealth, InValue))
	{
		UE_MVVM_BROADCAST_FIELD_VALUE_CHANGED(GetHealthRatio);
	}
}
void UStatusViewModel::SetMaxHealth(const float InValue)
{	
	if(UE_MVVM_SET_PROPERTY_VALUE(MaxHealth, InValue))
	{
		UE_MVVM_BROADCAST_FIELD_VALUE_CHANGED(GetHealthRatio);
	}
}
float UStatusViewModel::GetHealthRatio() const
{
	return CurrentHealth / MaxHealth;
}
```

`ViewModel` 에서는 `View`에서 필요한 데이터를 `Model` 로부터 받고, 가공하는 작업이 필요합니다. 코드는 최대 체력인 `MaxHealth` 와 현재 체력인 `CurrentHealth` , 그리고 둘의 비율을 반환하는 `GetHealthRatio` 함수가 있습니다. 

<br>

```cpp
/** If the property value changed then set the new value and notify. */
#define UE_MVVM_SET_PROPERTY_VALUE(MemberName, NewValue) \
	SetPropertyValue(MemberName, NewValueThisClass::FFieldNotificationClassDescriptor::MemberName)

template<typename T, typename U = T>
bool SetPropertyValue(T& Value, const U& NewValue, UE::FieldNotification::FFieldId FieldId)
{
	if (Value == NewValue) return false;

	Value = NewValue;
	BroadcastFieldValueChanged(FieldId);
	return true;
}
```

여기서 `UE_MVVM_SET_PROPERTY_VALUE` 매크로는 신규 데이터를 기존 데이터에 대입하는 역할을 하고, 변경이 감지되었다면 브로드캐스트로 뿌려줍니다. 

그리고 `UE_MVVM_BROADCAST_FIELD_VALUE_CHANGED` 는 특정 데이터가 변경됨에 따라 특정 함수를 호출하라고 알려주는 역할을 합니다. 쉽게 말하면, `CurrentHealth` 의 값이 변경되면 당연히 그에 따른 `Ratio` 의 비율 또한 변경되겠죠. 그래서 `GetHealthRatio` 을 갱신해야하다는 의미입니다.
    
<br>

### 2. ViewModel의 주입과 바인딩

여기서부터가 중요합니다. 위에서 언급한 3가지 타입에 따라 코드가 어떻게 달라지는지, 그리고 어떤 함정이 있는지 살펴보겠습니다.

---

#### Case A. Global View Model Collection (공유 인스턴스)

이 방식을 사용하면 `ViewModel`이 `UMVVMGameSubsystem`을 통해 전역적으로 관리됩니다.

##### 2-1. ViewModel 생성 및 수집

```cpp
if(UWorld* World = GetWorld())
{
	if (const UGameInstance* GameInstance = World->GetGameInstance())
	{
		if (UMVVMViewModelCollectionObject* Collection = GameInstance->GetSubsystem<UMVVMGameSubsystem>()->GetViewModelCollection())
		{
		  // 1. ViewModel 인스턴스 생성 및 설정
			StatusViewModel = NewObject<UStatusViewModel>(this, UStatusViewModel::StaticClass());				
			FMVVMViewModelContext UStatusViewModelContext = FMVVMViewModelContext(UStatusViewModel::StaticClass(), FName("StatusViewModel"));
			
			// 2. SubSystem에서 수집
			if(Collection->AddViewModelInstance(StatusViewModel , HUDViewModel)) return;
		}
	}
}
UE_LOG(LogTemp, Error, TEXT("ViewModel is not created properly"));
```

가장 먼저 필요한 ViewModel 인스턴스를 하나 생성하고, 이를 관리하는 `SubSystem` 인 `UMVVMGameSubsystem` → `UMVVMViewModelCollectionObject` 에 접근해서 생성한 `ViewModel`을 저장하도록 합니다. 이렇게 서브시스템에 저장해둬야 나중에 외부에서 쉽게 접근할 수 있습니다.

생성할 때, 이름은 원하는대로 작성하고 이후 위젯에서 ViewModel을 찾을 때, 서브시스템에서 이름을 기반으로 찾기 때문에 꼭 기억해두셔야 합니다.

> 파괴도 관리해야 되기 때문에 `ViewModel`의 생성을 담당하는 하나의 매니저에서 관리하는 것이 좋습니다.
{: .prompt-tip }

<br>

---

##### 2-2. 위젯에서의 ViewModel 설정

![Setting Global View Model Collection](/assets/img/post/MVVM/Global_Setting.png){: width="80%" style="display:block; margin: 0 auto;"}
*Setting Global View Model Collection*


이제 적용할 위젯으로 가서 위 그림과 같이 `Creation Type`을 `GlobalViewModelCollection`으로 `Identifier`을 아까 C++에서 지정했던 이름인 `StatusViewModel`으로 지정해줍니다.

<br>

---

##### ※ 주의 사항 - 자동의 허점

`GlobalViewModelCollection` 을 사용하면, 위젯이 생성되는 시점에 알아서 설정된 `ViewModel`을 찾아 바인딩을 진행하는데요. 여기서 중요한건 최초 1회만 진행을 한다는 점입니다.

`ViewModel`이 생성된 이후에 위젯이 생성된다면 정상적으로 바인딩이 진행될테지만, 순서가 바뀌어 위젯이 먼저 생성된다면, 바인딩이 진행되지 않을 겁니다. 생성 및 초기화 순서를 정확하게 하는게 가장 베스트죠.

<br>

#### Case B. CreateInstance (개별 인스턴스)

##### 2-1. 위젯에서의 ViewModel 설정

![Setting CreateInstance](/assets/img/post/MVVM/Create_Setting.png){: width="80%" style="display:block; margin: 0 auto;"}
*Setting CreateInstance*

위젯에서 ViewModel의 생성 타입을 `CreateInstance` 으로 지정해줍니다. 앞서 말했듯 위젯이 생성될 때, `PreConstruct`와 `Construct` 이벤트 사이에 자동으로 생성 및 바인딩 작업이 진행됩니다. 그렇기에 더 이상의 작업은 필요없죠.

> 다만 이렇게 되면, 소유권이 Widget에게 있기 때문에 MVVM 분리 원칙에는 맞지 않는 것 같네요.
{: .prompt-tip }

<br>

#### Case C. Manaul (개별 인스턴스)

##### 2-1. 위젯에서의 ViewModel 설정

![Setting Manaul](/assets/img/post/MVVM/Manual_Setting.png){: width="80%" style="display:block; margin: 0 auto;"}
*Setting Manaul*

위젯에서 ViewModel의 생성 타입을 `Manual` 로 지정해줍니다. 

<br>

---

##### 2-2. ViewModel 생성 및 지정

```cpp
// 1. Widget 생성 
NewWidget = CreateWidget<UUserWidget>(GetWorld(), NewWidgetClass);
NewWidget->AddToViewport();
		
// 2. ViewModel 생성 
UStatusViewModel* StatusViewModel = NewObject<UStatusViewModel>(this, UStatusViewModel::StaticClass());

// 3. ViewModel 지정
if (UMVVMView* View = Cast<UMVVMView>(Widget->GetExtension(UMVVMView::StaticClass())))
{
	FName VmName("StatusViewModel");
	if (View->GetViewModel(VmName) == nullptr)
	{
		View->SetViewModel(VmName, StatusViewModel);
	}
}
```

이번에는 직접 `ViewModel`을 생성하고, 위젯에 접근해서 `ViewModel`을 지정해주면 되겠습니다. 이 뷰모델 또한 관리될 필요가 있다면, 매니저 단에서 소유하는 것이 좋겠습니다.

<br>

---

### 3. 함수 바인딩

![Setting Function](/assets/img/post/MVVM/Function_Binding.png){: width="80%" style="display:block; margin: 0 auto;"}
*Setting Function*

ViewModel에서 `GetHealthRatio` 함수는 Max나 CurrentHealth가 변경될 때, 호출된다고 했죠? 이 함수와 Veiw에서의 함수를 바인딩해주면 `GetHealthRatio`가 호출될 때, 자동으로 해당 함수도 호출되며 뷰가 바뀔것입니다.

<br>

---

### 4. Model에서의 ViewModel 데이터 전달

```cpp
void UStatusViewModel::InitializeViewModel(AYourCharacter* Player)
{
	Player->OnMaxHealthUpdate.AddDynamic(this, &UStatusViewModel::SetMaxHealth);
	Player->OnCurrentHealthUpdate.AddDynamic(this, &UStatusViewModel::SetCurrentHealth);
}
```

이제는 `Model`에서 데이터 변경이 발생하면, 해당 이벤트와 데이터를 `ViewModel`에 전달할 수 있도록 연결해주어야 합니다.  위와 같이 `Model`의 이벤트와 `ViewModel`의 함수와 바인딩해두면 됩니다.  이렇게 되면 Health 관련 정보가 변경될 때, `ViewModel`의 데이터 또한 갱신되겠죠.

<br>

```cpp
// 잘못된 예시 (직접 호출)
void Player::SetMaxHealthValue(float InValue)
{
  StatusViewModel->SetMaxHealth(InValue);
}
```

여기서 중요한건 위와 같이 `Model`에서 `ViewModel`과 강한 연결이 되면 안된다는 겁니다. 그렇게 되면 유연한 연결을 위해서 사용하는 MVVM의 의미가 줄어들겠죠?

<br>

---

#### ※ 예외 - CreateInstance의 경우

```cpp
if (UMVVMView* view = Cast<UMVVMView>(Widget->GetWidget()->GetExtension(UMVVMView::StaticClass())))
{
	TScriptInterface<INotifyFieldValueChanged> VMInterface = view->GetViewModel(FName("StatusViewModel"));
	UStatusViewModel* StatusViewModel = Cast<UStatusViewModel>(VMInterface.GetObject());
	
	// 옳은 예시
	Player->OnMaxHealthUpdate.AddDynamic(StatusViewModel, &UStatusViewModel::SetMaxHealth);
	Player->OnCurrentHealthUpdate.AddDynamic(StatusViewModel, &UStatusViewModel::SetCurrentHealth);

  // 잘못된 예시 (직접 호출)
	StatusViewModel->SetMaxHealth(100.f);
}
```

생성 타입이 `CreateInstance` 인 경우에는 외부에서 해당 위젯의 `ViewModel` 에 대한 정보를 가지고 있지 않죠? 위 코드와 같이 위젯의 `Extension`에 접근해서 `ViewModel` 을 가져올 수 있습니다.

> 이때도 역시 직접 호출하게 되면 강한 의존성이 생기기 때문에 하면 안되겠습니다.
{: .prompt-warning }

<br>

### 5. 결과

![ToView Result](/assets/img/post/MVVM/Result_toView.gif){: width="80%" style="display:block; margin: 0 auto;"}
*ToView Result*

`Model`의 값이 바뀔 때 마다, `View`에 해당하는 체력바가 변경되는 모습을 볼 수 있습니다.

<br>

## 실습 - 체력 시스템 구현 (To Model)

그럼 이제 반대의 경우도 필요하겠죠. 위젯의 조작에 따라 데이터가 변경되는 경우입니다. 이렇게 되면 `View  → ViewModel → Model`로 데이터가 흘러가야 합니다.

가장 고민이 많았던 부분입니다. 단순히 함수를 호출하면 **무한 루프(Infinite Loop)**에 빠질 위험이 있습니다.

> View 변경 -> VM 업데이트 -> Model 업데이트 -> (Model Delegate) -> VM 업데이트 -> View 변경

### 1. ViewModel 확장

#### Case A. 새로운 함수 만들기

```cpp
/** Header */
UFUNCTION(BlueprintCallable)
void SetCurrentHealth(const float InValue);

UFUNCTION(BlueprintCallable)
void SetCurrentHealthFromView(const float InValue);

/** CPP */
void UStatusViewModel::SetCurrentHealthFromView(const float InValue)
{
	SetCurrentHealth(InValue);

	/** 직접 호출 */
	Player->SetCurrentHealth(InValue);
}
```

기존 `SetCurrentHealth` 는 `Model`에서 넘어온 데이터를 `View`에게만 데이터를 전달하는 방법이였죠. 그렇기 때문에 가장 간단한 방법은 위와같이 `View`에서 호출할 때 사용하는 함수를 만들어서 사용하는 방법입니다. 사용할 수 있지만, 이러면 함수가 많아지는 문제가 생기죠. 

> ViewModel에서는 어차피 Player 이벤트를 알고 있기 때문에 직접 호출했습니다.
{: .prompt-info }

<br>

---

#### Case B. 데이터 전달 위치 추가 - 추천

```cpp
USTRUCT(BlueprintType)
struct FViewModelFloat
{
	GENERATED_BODY()

	FViewModelFloat() :
		ViewModelUpdateSource(EViewModelUpdateSource::Normal),
		Value(0)
	{}
	FViewModelFloat(EViewModelUpdateSource InViewModelUpdateSource, float InValue) :
		ViewModelUpdateSource(InViewModelUpdateSource),
		Value(InValue)
	{}

	UPROPERTY(BlueprintReadWrite)
	EViewModelUpdateSource ViewModelUpdateSource;

	UPROPERTY(BlueprintReadWrite)
	float Value;

	bool operator==(const FViewModelFloat& Other) const
	{
		return ViewModelUpdateSource == Other.ViewModelUpdateSource && FMath::IsNearlyEqual(Value, Other.Value);
	}
	bool operator!=(const FViewModelFloat& Other) const
	{
		return !(*this == Other);
	}
	bool IsDefault() const
	{
		return ViewModelUpdateSource == EViewModelUpdateSource::Normal && FMath::IsNearlyZero(Value);
	}
};
```

그래서는 저는 열거형과 데이터가 포함된 구조체를 만들고, 해당 데이터가 어디서 들어왔는지 판단하도록 했습니다. 이렇게 하면, 하나의 함수를 사용하면서 데이터를 전달한 부분이 어디인지 알 수 있죠. 이렇게 하면 해당 구조체를 사용한다면, View에서 데이터를 수정할 수 있다는 것을 간접적으로 알 수 있습니다.

> 하지만 데이터 하나 추가할 때 마다, 구조체를 만들어야 한다는 부담이 생길 수 있을 것 입니다. "양방향 통신"이 필요한 경우에 대해서만 사용하는 것을 추천합니다.
{: .prompt-warning }

<br>

```cpp
void UStatusViewModel::SetCurrentHealth(const FViewModelFloat InViewModelFloat)
{
	if(UE_MVVM_SET_PROPERTY_VALUE(CurrentHealth, InViewModelFloat))
	{
		UE_MVVM_BROADCAST_FIELD_VALUE_CHANGED(GetHealthRatio);

		if(InViewModelFloat.ViewModelUpdateSource == EViewModelUpdateSource::FromView)
		{
			// Model에 접근 및 저장
			TargetPlayer->SetCurrentHealth(InViewModelFloat.Value);
		}
	}
}
```

기존 `SetCurrentHealth` 를 위와 같이 바꿔주고, `View` 에서의 데이터가 변경되면, 위 함수를 호출해서 `Model` 에게 전달해주면 됩니다.

<br>

---

### 2. 결과

![ToViewModel Result](/assets/img/post/MVVM/Result_toViewModel.gif){: width="80%" style="display:block; margin: 0 auto;"}
*ToViewModel Result*

이렇게 뷰포트를 통해서 입력한 정보가 `Model`의 `Current Health`에 저장되고, `View`인 체력 바가 변경되는 것을 볼 수 있습니다.

<br>

## 마무리

이렇게 C++를 이용해서 양방향 통신 작업하는 방법을 예시와 함께 설명해봤는데요. 설명이 잘 되었을지는 모르겠습니만, 확실히 MVVM을 구축하는 것은 초기 설정 비용이 꽤 들어가는 작업인 것 같습니다. 하지만 프로젝트 규모가 커질수록, 데이터의 흐름을 유연하게 통제할 수 있다는 점을 느끼실 수 있을 겁니다.

아무쪼록 저의 삽질이 여러분에게 큰 도움이 되었기를 바라며, 이번에 못 다뤘던 `Resolver`방식이나 `Property Path`등의 사용 방법이나, 경험 기반 팁들에 대해 다뤄보겠습니다.