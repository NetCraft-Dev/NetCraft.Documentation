# NetCraft 모딩 가이드


개별 API의 세부 사항은 여기 없습니다. [modapi-ko.md](modapi-ko.md)를 참고하세요.

---

## 1. 먼저 런타임 구조

### 1.1 세 개의 프로세스 진입점

NC에는 세 개의 진입점이 있으며, 모드 주입 파이프라인은 셋 모두 동일합니다:

| 진입점 | 용도 |
| --- | --- |
| `NetCraft.Loader` | 양쪽을 위한 하나의 exe: `--server`는 서버를 시작하고; `--client` 또는 모드 플래그 없음은 클라이언트를 시작 |
| `NetCraft.Server.Exe` | 독립 실행형 서버 실행 파일 |
| `NetCraft.Client.Exe` | 독립 실행형 클라이언트 실행 파일 |

`Main` 자체는 콜백만 등록하고 작업을 다음 메서드에 넘기는 얇은 껍데기입니다. `NetCraft.Server.Exe`를 예로 들면:

```csharp
public static int Main(string[] args)
{
    EmbeddedAssemblyLoader.Initialize();   // 커널 어셈블리 확인 콜백을 등록합니다
    BootMods(args);                        // 모드 부트스트랩을 실행합니다
    return Launch(args);                   // 이제서야 비즈니스 구현으로 들어갑니다
}
```

이 분리는 스타일의 문제가 아닙니다. JIT가 메서드를 컴파일할 때 메서드 본문에 나타나는 **모든 타입**을 해석하며, 이는 메서드가 실행되기 전에 일어납니다. 만약 `Main`이 `ServerMain.Run(args)`를 직접 호출하면 `Main`이 JIT 컴파일되는 순간 `NetCraft.Server.dll`이 끌어올려져, 모드 부트스트랩이 실행되기 전에 재작성 창이 사라집니다. 그래서 `BootMods`와 `Launch` 모두 `MethodImplOptions.NoInlining`으로 표시해야 합니다 — 표시가 없으면 JIT가 이들을 다시 `Main`으로 인라인하여 분리가 무력화됩니다.

`NetCraft.Loader`도 같은 구조인데, 모드 감지와 모드 부트스트랩이 둘 다 `Launch`에 있고 `Main`에는 `Initialize`와 `Launch` 두 단계만 남습니다.

### 1.2 kernel/ 하위 디렉터리의 커널 어셈블리

빌드 후 출력 디렉터리는 다음과 같습니다:

```
NetCraft.Server.Exe.exe
NetCraft.dll              <- 메인 라이브러리, 모든 하위 서브 라이브러리를 임베드함
NetCraft.ModLoader.dll    <- 로더 자체
NetCraft.Server.Exe.dll   <- 엔트리 어셈블리
kernel/
  NetCraft.Game.dll
  NetCraft.Server.dll
  ... 나머지 커널 어셈블리
mods/
  your-mod.dll
```

왜 루트에 두지 않고 `kernel/`로 옮길까요?

.NET 호스트는 `deps.json`에 등록된 어셈블리를 TPA(Trusted Platform Assemblies)로 취급합니다. TPA에 있는 어셈블리는 런타임이 **경로로** 확인하며 — `AssemblyLoadContext.LoadFromStream`에 전달된 바이트는 그냥 무시됩니다. 즉, 미리 재작성된 바이트를 넣어도 런타임은 디스크의 재작성되지 않은 복사본을 읽습니다. 커널 어셈블리를 `deps.json`에서 제거하고 파일을 다른 곳으로 옮겨야만, 확인 실패 시 런타임이 `AssemblyLoadContext.Resolving`으로 다시 콜백하여 재작성된 바이트를 넘겨줄 기회가 생깁니다.

루트에 남는 세 가지는 옮길 수 없습니다: 메인 라이브러리(임베딩 호스트이며 가장 먼저 시작해야 함), 로더 자체(부트스트랩 코드가 여기에 있음), 엔트리 어셈블리(apphost가 여기서 시작함)입니다.

**비용**: 엔트리 어셈블리 자체에는 주입할 수 없습니다. 훅 대상이 마침 `NetCraft.Server.Exe.dll` 어셈블리에 있으면 효과가 없습니다. 커널 비즈니스 코드는 모두 `kernel/` 아래에 있으므로 보통은 문제가 되지 않습니다.

### 1.3 모드 로딩 순서

```
EmbeddedAssemblyLoader.Initialize()
  └─ Resolving 콜백 설치
BootMods → ModBootstrap.Run(current side)
  ├─ mods/*.dll 정적 스캔 (MetadataReader가 임베드된 ncmod.json을 읽음, 어셈블리는 로드하지 않음)
  ├─ environment가 현재 측과 맞지 않는 모드 필터링
  ├─ 주입 규칙을 구성하고 재작성기를 메인 라이브러리에 전달
  ├─ 대상 어셈블리 프리로드: 바이트 읽기 → 재작성기 통과 → LoadFromStream
  └─ 각 모드 엔트리의 Init() 호출
Launch → ServerMain/ClientMain.Run(args)
  └─ 커널 비즈니스가 실행되기 시작함; 프로브는 이미 내부에 있음
```

순서에 주목하세요: **선언을 스캔하고, 재작성된 바이트를 로드하며, 엔트리 코드는 마지막에 실행됩니다**. 모드의 `Init()`이 실행될 때쯤이면 커널 어셈블리는 이미 교체되어 있습니다.

---

## 2. Fabric과의 주요 차이점

| 항목 | Fabric | NetCraft |
| --- | --- | --- |
| 언어 / 런타임 | Java / JVM | C# / .NET 10 (CoreCLR) |
| 모드 전달체 | `fabric.mod.json`을 담은 jar | `ncmod.json`을 임베드한 dll |
| 선언 읽기 | jar 내부의 파일 읽기 | `MetadataReader`가 어셈블리를 로드하지 않고 임베디드 리소스를 정적으로 읽음 |
| 코드 주입 | Mixin(소스의 애노테이션; 클래스 로드 시 멤버가 대상 클래스에 믹스인됨) | `Lead.Hook`(매니페스트나 애노테이션으로 선언된 규칙; 어셈블리 확인 중에 바이트가 제자리에서 재작성됨) |
| 주입 세분성 | 메서드 본문의 임의 줄, 지역 변수와 중간 표현식 값 포함 | 열세 가지 형태(호출 지점, 필드 읽기/쓰기, 생성자, 타입 검사, 박싱, 지역 변수, 상수, 메서드 본문 전체 교체, 프로브 등), 앞 또는 뒤 삽입 지원 |
| 로딩 모델 | Fabric Loader + Knot 클래스 로더 | 단일 기본 ALC + `AssemblyLoadContext.Resolving` |
| 공식 API 범위 | Fabric API는 모듈이 매우 많음 | NetCraft-ModApi는 현재 이벤트와 명령 확장 지점만 있음 |

### 2.1 주입 방식: 애노테이션 또는 매니페스트, 하나를 고르세요

Fabric의 Mixin은 **소스에서** 애노테이션됩니다:

```java
@Inject(method = "tick", at = @At("HEAD"))
private void onTick(CallbackInfo ci) { ... }
```

NC는 두 방식을 모두 지원하지만 전제 조건이 다릅니다: 애노테이션 방식은 `NetCraft.ModApi.Extension`의 `InjectAttribute`에 의존하므로 이를 참조하지 않는 모드는 애노테이션을 쓸 수 없습니다; 매니페스트 방식은 `ncmod.json`에 작성하는 순수 데이터이며 주입 규칙에 참조가 필요 없습니다.

**애노테이션**, 자신의 대체 메서드에 둡니다:

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick),
        HookType = "Mark", Label = "server_tick", Environment = "server")]
public static void OnTick(object self) { ... }
```

**매니페스트**, `ncmod.json`의 `hooks`에 작성합니다:

```json
{
  "target": "NetCraft.Game.Server.DedicatedServer",
  "method": "Tick",
  "type": "Mark",
  "label": "server_tick",
  "environment": "server",
  "replaceType": "MyMod.Probe",
  "replaceMethod": "OnTick"
}
```

어셈블리 구성 중에 두 경로가 하나의 규칙 테이블로 병합되며, **같은 주입 지점을 양쪽에 작성하면 애노테이션이 이깁니다**. 병합 후에는 구분할 수 없으며, 차이는 전제 조건과 사용 편의성에 있습니다:

| | 애노테이션 | 매니페스트 |
| --- | --- | --- |
| 작성 위치 | 대체 메서드 위 | `ncmod.json`의 `hooks` |
| 전제 조건 | `NetCraft.ModApi`를 참조해야 함 | 없음, 순수 데이터 |
| 타입 이름 | `typeof` / `nameof`, 컴파일러가 검사 | 손으로 쓴 문자열; 오타는 어셈블리 구성 중에만 발견됨 |
| 담을 수 있는 것 | 주입 규칙만 | 모드 정체성(id, entry, environment, 표시 정보)과 주입 규칙 |

따라서 애노테이션을 쓰든 안 쓰든 `ncmod.json`은 반드시 작성해야 합니다; 이것이 모드 정체성의 유일한 원천입니다. 애노테이션은 규칙을 실수 없이 작성하게 도와줄 뿐입니다. 매니페스트에는 종속성 필드가 없습니다; 종속성 관계는 어셈블리 참조에서 추론되며(6.1 참고) 선언할 필요가 없습니다.

반대로, **`NetCraft.ModApi`를 참조하지 않는 모드는 매니페스트만 사용할 수 있습니다** — 이는 주입 규칙 이상에 영향을 줍니다: `ServerEvents` 같은 이벤트와 `Nc*` 파사드도 ModApi(`NetCraft.ModApi.Wrapper` 아래, [3.3](#33-나란히-비교한-예제) 참고)에 있으므로, 애노테이션을 쓸 수 없는 모드는 이것들도 쓸 수 없습니다.

Fabric과의 차이는 여전히 남습니다:

- **변경이 일어나는 방식과 시점**: Mixin은 트랜스포머가 **클래스 로드** 시 **믹스인 클래스의 멤버를** 대상 클래스에 **믹스인**하므로, 로드되는 것은 합성된 새 클래스이고 원본은 더 이상 존재하지 않습니다; NC는 **어셈블리가 메모리에 들어가기 전에** 대상 메서드의 명령을 제자리에서 재작성하므로 클래스는 여전히 같은 클래스이고 메서드 본문만 바뀝니다. 둘 다 로드 시점에 재작성하며, 어느 쪽도 컴파일 시점에 바이트코드를 수정하지 않습니다 — Mixin의 애노테이션 프로세서는 빌드 시점에 refmap(난독화 매핑)만 생성하고 검증을 수행하는 반면, NC는 난독화되지 않으므로 그런 계층이 전혀 없습니다.
- Mixin은 **메서드 본문 중간의 임의 위치**에 주입할 수 있습니다; NC는 지정된 호스트 메서드 내의 특정 호출 지점, 필드 접근, 생성, 지역 변수 읽기/쓰기, 상수를 대상으로 할 수 있고 그 앞이나 뒤에 삽입할 수 있지만(`InType`/`InMethod`가 범위를 좁히고, `Placement`가 삽입 또는 교체를 결정), **임의의 줄 번호에 도달할 수 없고** 점프 대상이나 스택의 중간 표현식 값을 바꿀 수 없습니다.
- Mixin 대상은 문자열 메서드 이름과 디스크립터를 사용하고; NC는 "전체 타입 이름 + 메서드 이름"을 사용하므로 동명 오버로드가 모두 매칭되며, 하나로 정밀하게 지정하려면 `InType`/`InMethod`가 필요합니다.

**어느 계층이 애노테이션을 처리하는가**: 애노테이션 타입(`InjectAttribute`)은 `NetCraft.ModApi.Extension`이 제공하며, `NetCraft.ModLoader`가 해석합니다 — 모드를 스캔할 때 `MetadataReader`로 `CustomAttribute` 테이블을 정적으로 읽고 어셈블리는 로드하지 않습니다. **`Lead.Hook`은 애노테이션을 인식하지 못합니다**; 병합된 규칙 테이블만 보며, 네이티브 주입 계층은 관리 측에서 컴파일된 설명 바이트만 인식하고 `ncmod.json`은 읽지도 않습니다.

이것이 애노테이션이 표현할 수 있는 것을 결정합니다: 무엇을 쓸 수 있는지는 전적으로 `InjectAttribute`가 어떤 필드를 가지는지에 달려 있습니다. 현재 일곱 개가 있습니다 — 대상 타입, 메서드 이름, `HookType`, `Label`, `Environment`, `PatchMode`, `Ordinal` — 이며, [2.4](#24-단일-지점으로-좁히기-호스트-스코핑과-배치)의 `InType`/`InMethod`/`Placement`와 [2.5](#25-메서드-본문-내-앵커-지역-변수와-상수)의 `LocalIndex`/`ConstantValue`는 **애노테이션에 쓸 수 없습니다**; C# API를 사용하거나 매니페스트가 이를 지원할 때까지 기다리세요. 매니페스트 쪽도 이것들이 빠져 있습니다 — 애노테이션보다 더 받아들이는 것은 `ordinal`뿐입니다.

열세 가지 주입 형태는 [modding-guide 부록](#부록-hooktype-개요)과 [modapi-ko.md](modapi-ko.md)를 참고하세요.

### 2.2 중요한 제약: 프로브 클래스는 시그니처에 커널 타입을 담아서는 안 됩니다

NC 재작성은 커널 어셈블리가 로드되기 **전에** 일어납니다. 규칙을 구성할 때 `Lead.Hook`은 리플렉션으로 대체 메서드를 찾아 메서드 참조를 만들며, 이 과정에서 시그니처의 모든 매개변수 타입과 반환 타입을 해석합니다.

따라서: **대체 메서드의 시그니처는 BCL 타입과 `object`만 사용할 수 있습니다**. 시그니처에 `NetCraft.*` 타입이 나타나면 그것을 해석하면서 커널 어셈블리를 조기에 끌어올리고 주입이 아예 실패합니다.

커널 객체가 필요하면 매개변수를 `object`로 선언하고 메서드 본문 안에서 캐스트하세요:

```csharp
//어셈블리는 object만 보며, 메서드 본문은 커널이 시작된 후 JIT 컴파일됩니다
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    ...
}
```

### 2.3 CallSite는 삽입이 아니라 교체입니다

`CallSite` 규칙의 대체 메서드는 원본 호출을 **교체**하므로 원본 메서드는 실행되지 않습니다. 원래 동작을 유지하려면 대체 메서드에서 직접 복원해야 합니다:

```csharp
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    EffectCommand.Register(typed);              // 교체된 호출을 복원합니다
    ServerEvents.CommandRegister.Publish(...);  // 그런 다음 모드 자체 로직을 추가합니다
}
```

이 단계를 빠뜨리면 원래 기능이 완전히 사라집니다.

이를 수행하는 데 관한 몇 가지 세부 사항:

- **인스턴스 메서드의 `this`도 매개변수로 계산됩니다**. 대상 메서드가 인스턴스 메서드인지가 대체 메서드에 앞에 매개변수가 하나 더 필요한지를 결정합니다. IL에서 `call`과 `callvirt` 모두 인스턴스 호출로 계산됩니다 — `sealed` 타입의 비가상 메서드의 경우 컴파일러는 `call`을 내보냅니다.
- **하나의 규칙 키는 그 클래스 이름 아래의 모든 오버로드를 포괄합니다**; 매개변수 수가 같은 오버로드는 하나의 대체 메서드를 공유합니다. `Disconnect(string)`과 `Disconnect(Component)`가 이렇게 공유되며, 대체 메서드는 실제 인수 타입으로 디스패치합니다.
- **private 메서드는 외부에서 호출할 수 없으므로**, 대체 메서드가 원본 호출을 복원할 수 없습니다. 이 훅 지점을 포기하거나 리플렉션으로 한 번 호출하세요(빈도가 낮으면 허용됩니다).
- **값 타입 매개변수와 반환값은 `object`로 선언할 수 없습니다**, `object`는 스택에서 참조이고 `float`/`bool`은 값이며, 불일치는 유효하지 않은 IL입니다. 이 두 위치는 실제 타입으로 유지하세요.
- **유니캐스트 콜백은 직접 할당할 수 없습니다**. 일부 커널 콜백 속성(예: `ServerChunkCache`의 세 청크 콜백)은 `event`가 아니라 `Action<T>`이며, 커널이 이미 점유하고 있습니다. 모드가 직접 할당하면 아무 오류 없이 커널의 복사본을 덮어씁니다. 올바른 방법은 속성의 setter에 훅하여 할당 순간에 여러분의 로직과 커널 콜백을 하나의 래퍼 델리게이트로 연결하는 것입니다.

### 2.4 단일 지점으로 좁히기: 호스트 스코핑과 배치

명령 수준 규칙의 기본 범위는 **어셈블리 전체**입니다 — 대상 메서드를 호출하거나 대상 필드를 읽거나 쓰는 모든 곳이 매칭됩니다. 한 지점으로 좁히려면 두 개의 선택적 매개변수를 사용하세요:

| 매개변수 | 효과 |
| --- | --- |
| `InType` / `InMethod` | 지정된 호스트 메서드 본문 안에서만 앵커를 매칭함; 둘 다 비어 있으면 제한 없음 |
| `Placement` | `Replace`는 앵커를 교체함(기본값); `Before` / `After`는 앵커를 유지하고 그 앞이나 뒤에 호출 하나를 삽입함 |
| `Ordinal` | 같은 앵커가 호스트 메서드의 여러 곳과 매칭될 때 어느 것을 고를지, 0부터 시작. 생략하면 모든 곳이 수정됨 |

```csharp
//예: LevelChunk가 블록 상태를 읽을 때만 계측하고, 다른 곳의 PalettedContainer::Get은 건드리지 않음
new HookRule("NetCraft.Storage.PalettedContainer", "Get", typeof(MyProbe), nameof(MyProbe.OnGet),
    HookType.CallSite, PatchMode.ILRewrite,
    inType: "NetCraft.Storage.LevelChunk", inMethod: "GetBlockState",
    placement: HookPlacement.Before)
```

두 모드는 콜백 시그니처에 서로 다른 요구 사항을 부과합니다:

- **Replace 모드**는 교체된 호출의 인수와 정렬됩니다(인스턴스 호출의 경우 `this` 포함); 콜백이 원본 호출을 복원할지는 여러분에게 달려 있습니다.
- **Insert 모드**는 **호스트 메서드의 매개변수**를 전달하며(`this` 포함), `MethodBody` 규약과 일치합니다. 삽입은 앵커가 이미 쌓아 놓은 스택을 방해하지 않으며, 원본 호출은 평소처럼 실행되고 앞이나 뒤에 콜백이 하나 추가될 뿐입니다.

몇 가지 경계:

- `InType`과 `InMethod`는 독립적이며 하나만 지정해도 됩니다. 둘 다 비어 있는 것은 스코핑이 없는 것과 같습니다.
- 여러 규칙이 같은 앵커에 훅할 수 있으며 각각 다른 호스트로 스코핑됩니다; **첫 번째 호스트 매칭이 이깁니다**.
- `Placement`는 명령 수준 형태(`CallSite`, `NewObj`, 필드 읽기/쓰기, `TypeCheck`, `Box`, `FunctionPointer`, 그리고 [2.5](#25-메서드-본문-내-앵커-지역-변수와-상수)의 세 종류)에만 적용됩니다; `MethodBody`는 항상 전체를 교체합니다.
- `Ordinal`은 그 지점이 최종적으로 수정되는지와 무관하게 **매칭 순서**를 셉니다; 호스트 메서드에서 규칙이 충분히 여러 번 나타나지 않으면 규칙이 적용되지 않습니다. Mixin의 `@At(ordinal)`과 같은 개념입니다.
- `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue`는 현재 C# API에서만 사용할 수 있습니다; `ncmod.json`과 `[Inject]` 모두 이를 지원하지 않으므로(매니페스트는 `ordinal`을 받음), 매니페스트 기반 모드는 앞의 몇 가지를 쓸 수 없습니다.

### 2.5 메서드 본문 내 앵커: 지역 변수와 상수

앞의 종류들은 **참조된 엔티티**(메서드, 필드 또는 생성자)에 앵커링하는 반면, `LocalRead` / `LocalWrite` / `Constant`는 **호스트 메서드 본문 내의 위치**에 앵커링하며, Mixin의 `@ModifyVariable`과 `@ModifyConstant`에 대응합니다. 이 세 가지의 경우 `OriginalType` / `OriginalMethod`는 참조된 엔티티가 아니라 **호스트 메서드**를 지정합니다.

| 형태 | 추가 매개변수 | 선택 위치 |
| --- | --- | --- |
| `LocalRead` | `LocalIndex` | 그 슬롯의 모든 읽기(0부터 시작) |
| `LocalWrite` | `LocalIndex` | 그 슬롯에 대한 모든 쓰기 |
| `Constant` | `ConstantValue` | 그 상수의 모든 로드, 박싱된 타입으로 비교 |

```csharp
//예: G의 슬롯 0에 대한 쓰기 전에 콜백 하나를 삽입
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnWrite),
    HookType.LocalWrite, PatchMode.ILRewrite,
    localIndex: 0, placement: HookPlacement.Before)

//예: G의 상수 5를 OnConst()의 반환값으로 교체
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnConst),
    HookType.Constant, PatchMode.ILRewrite, constantValue: 5)
```

replace 모드에서 콜백 시그니처는 호스트 매개변수가 아니라 **명령의 스택 효과**와 정렬됩니다:

| 앵커 | 스택 효과 | 대체 메서드 시그니처 |
| --- | --- | --- |
| `LocalRead` | 값을 하나 푸시함 | 매개변수 없음, 그 값을 반환 |
| `LocalWrite` | 값을 하나 팝함 | 매개변수 하나 |
| `Constant` | 값을 하나 푸시함 | 매개변수 없음, 그 값을 반환 |

`ConstantValue`는 박싱된 타입으로 비교되므로 `5`(int)와 `5L`(long)은 서로 다른 앵커입니다; `ldc.i8`을 매칭하려면 `long`을 전달해야 합니다.

슬롯은 컴파일된 지역 변수 인덱스입니다; 같은 소스라도 컴파일러 버전이 다르면 슬롯이 바뀔 수 있으므로, 버전 간 이식할 때 안정적인 식별자로 취급하지 마세요.

### 2.6 런타임 주입: 이미 실행 중인 코드 수정

지금까지 논의한 주입은 모두 **어셈블리 로드 전에** 일어납니다 — 먼저 바이트를 재작성한 다음 런타임에 넘깁니다. 전제는 대상 어셈블리가 아직 로드되지 않았다는 것입니다.

`Lead.Hook`에는 또 다른 경로가 있습니다: CLR의 Profiler 인터페이스(ReJIT)를 사용하여 **이미 로드되었거나 메서드가 이미 실행된** 코드를 수정합니다. 둘 다 같은 `HookRule`을 공유하며, [2.4](#24-단일-지점으로-좁히기-호스트-스코핑과-배치)와 [2.5](#25-메서드-본문-내-앵커-지역-변수와-상수)의 매개변수를 계속 사용할 수 있습니다:

```csharp
var engine = new HookEngine();
engine.AddRule(new HookRule("TargetLib.Host", "Callee", typeof(Hooks), nameof(Hooks.Double),
    hookType: HookType.CallSite, patchMode: PatchMode.RuntimeInject, inMethod: "A"));

//호출 한 번으로 주입이 완료됨; 재시작도 파일 변경도 없음
RuntimeInjector.Inject(typeof(Host).Assembly, engine);
```

| | 로드 시 재작성 | 런타임 주입 |
| --- | --- | --- |
| 매니페스트 `patchMode` | `ILRewrite` (기본값) | `RuntimeInject` |
| 시점 | 어셈블리가 메모리에 들어가기 전 | 프로세스가 시작된 후 언제든 |
| 기반 | Mono.Cecil 바이트 재작성 | CLR Profiler ReJIT |
| 전제 조건 | 대상이 로드되지 않았음 | 대상이 이미 프로세스에 있음 |
| 이미 JIT 컴파일된 코드 수정 | 불가능 | 가능 |

**왜 규칙을 공유하는가**: 여기서도 Cecil이 재작성을 수행하지만 결과를 디스크에 쓰지 않고, 네이티브 계층에 넘길 설명으로 컴파일하여 런타임에 새 메서드 본문을 CLR에 제출하며, 나머지 버전 관리는 CLR에 맡깁니다.

**모드는 어떻게 하는가**: `ncmod.json`의 규칙 항목에 `patchMode`를 추가하세요; `[Inject]` 애노테이션에도 같은 이름의 매개변수가 있습니다.

```json
{ "target": "NetCraft.Game.Server.DedicatedServer", "method": "Tick",
  "type": "CallSite", "patchMode": "RuntimeInject",
  "replaceType": "MyMod.Probe", "replaceMethod": "OnTick" }
```

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick), PatchMode = "RuntimeInject")]
public static void OnTick(object self) { }
```

**어셈블리(구성)가 하는 일**: 이 규칙들은 로드 시 재작성 경로를 거치지 않습니다(`ModHooks.Rewrite`는 `ILRewrite`만 적용합니다); 어셈블리 구성 시 별도의 런타임 대상 테이블에 기록됩니다. 커널 어셈블리를 프리로드한 후 모드의 `Init()` 전에, 로더가 각 모드의 **이미 로드된** 인스턴스를 가져와 재작성된 메서드 본문을 CLR에 제출합니다. 그 시점에 대상이 로드되지 않았으면 경고와 함께 건너뛰며, 대신 조기에 로드하지도 않습니다 — [2.2](#22-중요한-제약-프로브-클래스는-시그니처에-커널-타입을-담아서는-안-됩니다)의 "커널을 조기에 끌어올리면 창을 놓친다"는 제약이 여기서는 방향이 반대이지만 결론은 같습니다: 없으면 할 수 없습니다.

**네이티브 주입 계층에 훅하려면**: ReJIT 스위치는 프로세스 시작 시 환경 변수(`CORECLR_ENABLE_PROFILING` / `CORECLR_PROFILER` / `CORECLR_PROFILER_PATH`)로만 설정할 수 있으며, 시작 후에 설정하면 효과가 없습니다. 로더가 시작 초기에 `RuntimeInject` 규칙을 감지했는데 프로세스가 아직 훅되지 않았으면, 이 세 변수를 실은 **같은 명령줄로 프로세스를 재시작**합니다(`NC_PROFILER_ATTACHED=1`은 "훅되었지만 효과 없음"일 때 반복 재시작을 방지합니다). 네이티브 라이브러리 `lead_hook_native`는 프로그램 루트에 두거나 `NC_PROFILER_PATH`로 다른 곳을 가리켜야 합니다; 둘 다 없으면 규칙 묶음 전체가 경고 하나로 강등되고 시작은 막히지 않습니다.

**같은 대상에 로드 시 재작성과 섞지 마세요**: 런타임 주입이 제출한 메서드 본문은 **원본 바이트**에서 파생되며, 같은 메서드에 로드 시 재작성이 가한 변경을 포함하지 않습니다 — 같은 메서드가 두 규칙 유형에 모두 걸리면 로드 시 버전이 완전히 덮어써집니다. 구성 단계에서는 두 규칙이 같은 메서드에 걸리는지 알 수 없으므로 대상 어셈블리로 거친 판단을 하고 경고를 기록할 수밖에 없습니다.

**애노테이션은 이 경로에 직접 관여하지 않습니다**: 네이티브 주입 계층은 `InjectAttribute`를 인식하지도, `ncmod.json`을 읽지도 않으며 설명 바이트만 인식합니다. 애노테이션과 매니페스트는 모두 **어셈블리 구성 시점**의 것이며([2.1](#21-주입-방식-애노테이션-또는-매니페스트-하나를-고르세요) 참고), `NetCraft.ModLoader`가 `HookRule`로 파싱한 뒤 `RuntimeInjector`에 넘깁니다; 모드 쪽에서 다르게 처리할 필요는 없습니다.

**제한 사항**(로드 시 재작성보다 좁음): 예외 처리 테이블이 있는 메서드는 지원하지 않고, 지역 변수 테이블은 변경할 수 없으며, 제네릭 타입과 메서드는 지원하지 않고, 피연산자는 메서드 참조만 인식합니다(필드 참조, 문자열 상수, 타입 토큰은 `NotSupportedException`을 던집니다).

**성능**: 주입은 등록 시에만 일어나며, 이후 메서드는 주입이 없을 때와 호출 오버헤드가 같은 평범한 JIT 코드입니다. 프로파일러 부착에는 일회성 비용이 있습니다 — ReJIT를 활성화하려면 ReadyToRun 이미지를 동시에 비활성화해야 하므로 측정된 프로세스 시작이 약 80~110ms 느려지지만, 정상 상태 계산에는 차이가 없습니다. 시작이 이미 초 단위로 측정되는 NC 같은 서버에서는 무시할 만합니다.

### 2.7 같은 클래스를 수정하는 두 모드가 Mixin처럼 충돌합니까?

먼저 Mixin 쪽에서 왜 충돌하는지 알아봅시다. Mixin은 **멤버를 대상 클래스에 믹스인**하고 클래스 로드 시 적용합니다: 여러 믹스인이 같은 클래스에 믹스인될 때, 같은 곳에 반복 주입하거나 같은 클래스에 동명 멤버를 추가하는 경우 `MixinApplyError`가 발생하며, 기본 fail-hard 동작은 **게임을 곧바로 종료시킵니다**; 게다가 이 감지는 클래스 로드 순간에 일어나므로 게임이 이미 절반쯤 실행 중일 수 있습니다.

NC의 모델은 다르며, 충돌이 일어날 수 있는 표면이 훨씬 작습니다:

| | Mixin | NetCraft |
| --- | --- | --- |
| 적용 방식 | 대상 클래스에 멤버 믹스인 + 바이트코드 재작성 | IL 명령만 재작성; 타입 합성 없음, 멤버 추가 없음 |
| 구조적 충돌(동명 멤버, 상속 충돌) | 있음 | 없음 |
| 규칙 검증 시점 | 클래스 로드 시 | 어셈블리 구성 시, 메타데이터를 정적으로 읽음 |
| 같은 곳에 걸린 두 규칙 | 예외 발생 | 선착순; 후자는 조용히 실패 |
| 한 모드 실패 시 | 전체 로드를 망칠 수 있음 | 자기 자신에게만 영향 |

**정적 검증**: 규칙은 어셈블리를 로드하고 타입을 리플렉션하여 만들어지는 것이 아니라 PE 메타데이터 테이블을 읽어 만들어집니다. 따라서 "대상 타입이 알려진 어셈블리에 없음"이나 "주입 형태 오타" 같은 문제는 클래스가 로드될 때까지 기다렸다 터지지 않고 **시작 초기에** 기록되고 건너뜁니다.

**실패 격리**: 모드의 규칙 파싱이 실패하거나, 대체 클래스 로드가 실패하거나, 엔트리 `Init()`이 예외를 던지면 **그 모드 하나만** 실패로 표시되고(상태 `Error`, MODS 페이지에 "load failed"로 표시됨) 다른 모드는 평소처럼 로드됩니다. 여기서 표현 하나를 바로잡아야 합니다: NC에는 **런타임 언로드가 없습니다** — 모드는 한 번 로드되며 `ModManager`는 동적 로드/언로드를 명시적으로 제공하지 않습니다. 이른바 "실패 시 자동 언로드"는 사실 **로드 시 격리**입니다: 실패한 모드는 초기화되지 않지만 "언로드"되는 것도 아닙니다.

**동적 주입**: [2.6](#26-런타임-주입-이미-실행-중인-코드-수정)의 ReJIT 경로에서 여러 모드가 같은 메서드를 두고 경쟁하는 의미론은 정적 경우와 같습니다 — 먼저 등록한 쪽이 이기고, 나중 요청은 전송되지만 차지할 수 없습니다(`GetReJITParameters`는 "모듈 + 메서드"로 차지하며 첫 번째를 취합니다).

이 경로에서 한 번 함정을 만났는데 기록할 만합니다: 초기 구현에서 `FindTypeRef`의 **resolution scope 매개변수를 `mdTokenNil`로 전달**했는데, 그 의미는 "resolution scope가 없는 TypeRef만 매칭"입니다 — 우리 참조는 모두 `AssemblyRef`에 매달려 있어서 만들어 둔 것이 결코 발견되지 않았습니다. 증상은 이러했습니다: 첫 주입은 성공했지만 두 번째 주입에서 참조 해석이 실패하고 새 메서드 본문을 만들 수 없었으며, CLR이 원본 IL로 되돌아가면서 **첫 주입까지 함께 사라졌습니다**(대상 메서드가 주입되지 않은 동작으로 되돌아감). 수정 후에는 재현되지 않았지만 제한은 남아 있습니다: **참조의 메타데이터 주입은 대상 모듈이 로드된 직후의 창 안에서 완료되어야 하며, 늦을수록 실패할 가능성이 큽니다**.

**비용은 분명히 말해야 합니다**: NC의 비충돌 동작은 충돌을 놓치기 쉬운 대가를 치릅니다 — Mixin은 최소한 로딩을 중단하지만, NC는 후자를 조용히 실패하게 둡니다. 이를 해결하기 위해 구성 단계에서 **동일 앵커 충돌 검사**를 수행합니다: 같은 주입 지점이 여러 모드에 선언되면 나중에 구성된 쪽을 `ModHooks.Warnings`에 기록하고 시작 로그에 경고로 보고합니다(로딩을 막지 않음):

```
Mod injection conflict mod my-mod-b's injection NetCraft.Game.Server.DedicatedServer::Tick[CallSite/ILRewrite] is already taken by mod my-mod-a; this rule will not take effect
```

감지 키는 "대상 타입 + 메서드 + 주입 형태 + 패치 모드"입니다. **호스트 스코핑은 구분하지 않습니다** — 매니페스트도 애노테이션도 `InType`/`InMethod`를 쓸 수 없으므로 모드에서 오는 규칙은 자연히 어셈블리 전체 범위이고, 같은 키면 충돌입니다. C# API로 직접 추가한 규칙은 이 검사를 우회합니다: 이 경우 두 규칙이 각각 다른 호스트에 걸릴 수 있어 본질적으로 충돌하지 않기 때문입니다.

**런타임 주입도 이 검사를 거칩니다**: 같은 구성 진입점으로 들어가며, 감지 키의 패치 모드가 이를 로드 시 재작성과 분리합니다; 두 유형이 같은 대상 어셈블리에 적용되면 별도의 덮어쓰기 알림이 있습니다([6.4](#64-같은-대상을-주입하는-두-모드) 참고).

### 2.8 믹스인: 대상 타입에 멤버 추가

앞 절들은 모두 기존 코드의 명령을 수정하며 새로운 것을 만들어 낼 수 없습니다. 대상 타입에 **필드, 메서드 또는 인터페이스를 추가**하려면 믹스인을 사용하세요.

Mixin 문법과의 관계는 다음과 같습니다:

| Mixin | NC |
| --- | --- |
| 믹스인 클래스의 `@Mixin(X.class)` | 소스 클래스의 `[Mixin(typeof(X))]` |
| 믹스인 클래스의 멤버가 대상 클래스에 믹스인됨 | 소스 클래스의 필드와 메서드가 대상 타입으로 이동됨 |
| `@Unique`가 private 필드 추가 | 소스 클래스에 일반 필드를 작성; 같은 방식으로 이동됨 |
| `@Shadow`가 대상 클래스의 기존 멤버 참조 | 필요 없음; `X`의 멤버를 직접 작성한 뒤 훅하면 됨 |
| `@Implements` / `implements` | `Interfaces` |

매니페스트에도 작성할 수 있습니다:

```json
{
  "target": "NetCraft.Game.World.Entity.SomeEntity",
  "source": "MyMod.SomeEntityMixin",
  "interfaces": ["MyMod.ITagged"]
}
```

```csharp
[Mixin(typeof(SomeEntity), Interfaces = new[] { typeof(ITagged) })]
public class SomeEntityMixin
{
    //이동된 후에는 대상 타입의 인스턴스 필드입니다
    public int MyCounter = 5;

    //믹스인된 메서드; 함께 이동한 필드를 읽고 씁니다
    public int Bump() => MyCounter + 1;

    //인터페이스가 요구하는 구현; 이동된 후 대상 타입이 ITagged를 구현합니다
    public string Describe() => $"tagged:{MyCounter}";
}
```

**복사가 아니라 이동입니다**. 이 멤버들은 소스 클래스에서 제거되어 빈 껍데기만 남습니다 — Mixin의 믹스인 클래스와 마찬가지로 **모드 코드는 더 이상 그 클래스를 사용해서는 안 됩니다**(`new SomeEntityMixin()`를 하거나 그 메서드를 호출하면 멤버를 찾지 못합니다).

몇 가지 적용 규칙:

- **로드 시 재작성에만 적용됩니다**. 대상 타입은 아직 메모리에 들어가지 않은 커널 어셈블리나 모드 어셈블리에 있어야 합니다. 런타임 주입은 메서드 본문만 제출하고 타입 레이아웃을 바꿀 수 없으므로 이 형태는 런타임 주입에 존재할 수 없습니다.
- **생성자 초기화도 함께 갑니다**. 소스 클래스의 필드 초기화에 작성된 값은 대상 타입의 모든 인스턴스 생성자에 병합되고, 정적 필드 초기화는 정적 생성자에 병합됩니다(대상에 없으면 생성됨). 소스 클래스 생성자 내부의 기본 클래스 연결 호출은 제거되므로 기본 생성자가 두 번 실행되지 않습니다.
- **인터페이스가 있는 규칙은 이동된 public 인스턴스 메서드를 virtual로 표시합니다**. 인터페이스 디스패치는 vtable만 인식하므로, 표시가 없으면 CLR이 인터페이스가 구현되지 않았다고 판단하여 로드 자체가 실패합니다. 따라서 인터페이스를 믹스인할 때 그 메서드들이 비가상으로 남기를 기대하지 마세요.
- **동명 멤버는 건너뜁니다**. 대상 타입에 이미 같은 이름의 필드나 메서드가 있으면 그 항목 하나는 이동되지 않고 나머지는 평소처럼 진행됩니다. 두 모드가 같은 대상 타입에 믹스인하면 둘 다 적용되며, 후자의 이름이 충돌하는 부분만 건너뜁니다 — 이는 [2.7](#27-같은-클래스를-수정하는-두-모드가-mixin처럼-충돌합니까)의 훅에서 "후자가 완전히 실패"하는 동작보다 완만합니다.
- **중첩 타입은 이동되지 않으며**, 소스 클래스의 중첩 타입이나 제네릭 메서드도 현재 이 경로의 범위 밖입니다.

소스 타입은 **모드 자체 어셈블리**에 있어야 하므로, 매니페스트도 애노테이션도 어셈블리 이름을 작성하지 않습니다.

### 2.9 두 가지 경로: 래퍼 계층과 확장 지점

`NetCraft.ModApi`의 공개 표면은 두 개의 네임스페이스로 나뉘며, 두 가지 용법에 대응합니다:

| 네임스페이스 | 내용 | 얻는 것 |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | `ServerEvents` / `ClientEvents` / `NetworkEvents`, `NcServer` / `NcWorld` / `NcPlayers` / `NcLists` / `NcRegistries` / `NcRecipes` / `NcStartup`, 그리고 `NcPlayer` / `NcLevel` 같은 객체 핸들 | 래퍼 타입; 공개 표면에 커널 타입이 없음 |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 애노테이션 | 규칙이 커널 클래스와 메서드 이름에 결합됨 |

둘은 **나란히 존재하는** 경로이며, 하나가 다른 하나 위에 계층을 이루는 것이 아닙니다:

- **안정성을 위해 `Wrapper`를 사용하세요**. 파사드가 커널의 번거로운 호출 순서를 대신 처리해 주며(레벨, 플레이어 목록, 동기화 체인을 건드리면서 블록 하나를 쓰는 것이 한 예입니다), 이벤트 args는 모두 래퍼 타입입니다. 대가는 파사드가 노출하지 않는 기능은 사용할 수 없다는 점입니다.
- **완전성을 위해 `Extension`을 사용하세요**. 주입 규칙이 커널 클래스와 메서드를 직접 수정하지만, 작성하는 대상 이름이 커널의 이름이므로 커널이 바뀌면 규칙도 함께 바뀌어야 합니다.

둘 다 참조할 수 있습니다. `Wrapper` 계열은 "공개 표면에 커널 타입이 없음"을 향해 통합되고 있으며, 플레이어와 레벨 부분은 완료되었습니다 — 플레이어 이벤트의 `Player` / `Attacker`와 `NcPlayers`의 입/출력 매개변수는 `NcPlayer` 핸들이고, `NcWorld.Overworld` / `Nether` / `End` / `Get`과 `LevelTickArgs.Level`은 `NcLevel` 핸들이며, 블록 좌표는 평범한 `x y z` 정수입니다; 엔티티와 나머지 값 타입(`BlockPos` / `BlockState` / `Vec3`)은 아직 감싸지지 않았습니다.

한 가지 경계를 더 짚어야 합니다: **래퍼 계층이 주입을 보호하지는 않습니다**. 작성한 `hooks` 규칙이나 `[Inject]` 애노테이션은 여전히 커널 클래스와 메서드 이름에 결합되며, 커널이 바뀌면 마찬가지로 깨집니다.

---

## 3. Fabric에서 마이그레이션

### 3.1 개념 매핑

| Fabric | NetCraft |
| --- | --- |
| `fabric.mod.json` | dll에 임베드된 `ncmod.json` |
| `ModInitializer.onInitialize()` | 엔트리 클래스의 `public Task Init()` |
| `@Inject` / `@Redirect` | `[Inject]` 애노테이션, 또는 `hooks`의 `Mark` / `Probe` / `CallSite` 같은 규칙 |
| `@ModifyVariable` | `LocalRead` / `LocalWrite`, [2.5](#25-메서드-본문-내-앵커-지역-변수와-상수) 참고; 애노테이션으로 작성 불가 |
| `@ModifyConstant` | `Constant`, [2.5](#25-메서드-본문-내-앵커-지역-변수와-상수) 참고; 애노테이션으로 작성 불가 |
| `@Accessor` | 아직 대응 없음(`private` 멤버는 가시성 확장이 필요 없으며 규칙을 작성하면 됨) |
| `Registry.register(...)` | 커널 레지스트리(`BuiltInRegistries`) |
| `ServerLifecycleEvents.SERVER_STARTED` | `ServerEvents.Started` |
| `ServerTickEvents.END_SERVER_TICK` | `ServerEvents.Tick` |
| `CommandRegistrationCallback` | `ServerEvents.CommandRegister` |
| `ClientTickEvents.END_CLIENT_TICK` | `ClientEvents.Tick` |
| `FabricLoader.getInstance().getModContainer(id)` | 아직 대응 없음(`ModManager`는 모드에 개방되지 않음) |
| `@Mixin` / `@Unique` / `@Implements` | `[Mixin]` 애노테이션 또는 매니페스트의 `mixins`, [2.8](#28-믹스인-대상-타입에-멤버-추가) 참고 |

### 3.2 이어지지 않는 것

- **Mixin의 애노테이션 시스템**: NC에는 `[Inject]`와 `[Mixin]` 두 애노테이션이 있으며, 둘 다 **선언 방식**일 뿐이고 `ncmod.json`의 `hooks` / `mixins`와 동등하며 어셈블리 구성 시 병합됩니다(이는 `Lead.Hook`이 아니라 로더가 해석합니다, [2.1](#21-주입-방식-애노테이션-또는-매니페스트-하나를-고르세요) 참고). 명령 재작성은 기본적으로 로드 시점에 적용되며 [2.6](#26-런타임-주입-이미-실행-중인-코드-수정)에 따라 런타임 제출로 바꿀 수 있습니다; 멤버와 인터페이스 추가는 [2.8](#28-믹스인-대상-타입에-멤버-추가)의 믹스인을 거칩니다. 대상 지정에는 `@At` 스타일 문자열 문법이 없지만, `CallSite`/`FieldRead`/`LocalWrite`/`Constant` 같은 형태와 `Ordinal`, `InType`/`InMethod`, `Placement`가 `HEAD`/`RETURN`/`INVOKE`/`FIELD`/`NEW`/`CONSTANT`/`LOAD`/`STORE` 및 `shift`의 용법을 커버할 수 있습니다; 빠진 것은 `JUMP`입니다.
- **AccessWidener**: 없음. NC에서 가시성은 IL 재작성에 장애가 되지 않습니다; `private` 메서드도 같은 방식으로 훅할 수 있습니다(재작성기는 바이트 수준에서 동작합니다).
- **Yarn / Mojang 매핑**: 필요 없음. NC는 바닐라에서 직접 번역한 C# 소스이며, 타입과 메서드 이름이 바닐라에 대응하고 명명 스타일만 C#을 따릅니다.
- **대부분의 Fabric API 모듈**: `NetCraft-ModApi`가 포괄하는 기능만 사용할 수 있으며, 나머지는 직접 주입 규칙을 작성하거나 API가 따라잡을 때까지 기다려야 합니다.

### 3.3 나란히 비교한 예제

Fabric: 서버가 시작될 때 한 줄을 로그하고 명령을 등록합니다.

```java
public class MyMod implements ModInitializer {
    @Override
    public void onInitialize() {
        ServerLifecycleEvents.SERVER_STARTED.register(server -> {
            System.out.println("server started");
        });
        CommandRegistrationCallback.EVENT.register((dispatcher, registry, env) -> {
            dispatcher.register(CommandManager.literal("mymod")
                .executes(ctx -> { ctx.getSource().sendSuccess(() -> Text.literal("hi"), false); return 1; }));
        });
    }
}
```

NC:

```csharp
public sealed class MyModEntry
{
    public Task Init()
    {
        ServerEvents.Started.Subscribe(_ => Log.Info("server started"));
        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    return 1;
                })));
        return Task.CompletedTask;
    }
}
```

매니페스트(`ncmod.json`, 임베디드 리소스로):

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "environment": "server",
  "entry": "MyMod.MyModEntry",
  "hooks": []
}
```

참고: 여기서는 빈 `hooks`도 작동합니다 — `ServerEvents.Started` 같은 이벤트는 `NetCraft-ModApi` 자체 프로브가 제공하며, 모드는 구독만 하면 됩니다(`ServerEvents`는 `NetCraft.ModApi.Wrapper` 아래에 있습니다, [2.9](#29-두-가지-경로-래퍼-계층과-확장-지점) 참고). ModApi가 아직 이벤트를 제공하지 않는 커널의 한 지점을 훅하고 싶을 때만 직접 훅 규칙을 작성하면 됩니다.

---

## 4. ncm의 기본 요구 사항

ncm은 NetCraft 모드를 뜻합니다. ncm은 `ncmod.json`을 임베드한 .NET 클래스 라이브러리 dll이며, `mods/` 디렉터리에 둡니다.

가장 쉬운 시작 방법은 템플릿입니다:

```
dotnet new install NetCraft.ModsProjectType
dotnet new ncm -n MyMod -e server
```

로컬 패키지가 아직 게시되지 않았으면 `dotnet new install <nupkg path>`를 사용하거나, 저장소에서 `dotnet pack`을 실행한 뒤 출력을 설치하세요.

`-e`는 `both`(기본값) / `server` / `client`를 받으며, 매니페스트의 `environment`와 엔트리 클래스에 생성되는 측의 구독 코드를 결정합니다. 템플릿은 NC 참조 어셈블리를 함께 제공하므로 프로젝트 참조가 필요 없고, `ncmod.json`의 id / entry는 프로젝트 이름에서 채워집니다.

4.1부터는 직접 작성한 프로젝트가 충족해야 하는 것을 다룹니다.

### 4.1 프로젝트 파일

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Private를 꺼야 합니다. 그렇지 않으면 4.6의 EmbedDependencies가 NC 어셈블리를 모드 dll에 임베드합니다 -->
    <ProjectReference Include="xxx\NetCraft\NetCraft.csproj" Private="false" />
    <ProjectReference Include="xxx\NetCraft.ModApi\NetCraft.ModApi.csproj" Private="false" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="ncmod.json" LogicalName="ncmod.json" />
  </ItemGroup>
</Project>
```

`LogicalName`은 `ncmod.json`이어야 합니다; 스캐너는 그 이름만 인식합니다.

템플릿은 다른 경로를 취합니다: `libs/` 아래의 어셈블리 참조(`Reference Include="libs\*.dll" Private="false"`)입니다. NC 소스가 없을 때 이 방식을 따르세요. 두 접근 방식의 공통점은 **NC 자체 dll이 출력 디렉터리에 절대 들어가서는 안 된다**는 것입니다 — [4.6](#46-서드파티-종속성)의 `EmbedDependencies`가 출력 디렉터리의 서드파티 dll을 모드에 임베드하는데, NC 어셈블리까지 임베드되면 두 세트의 타입 정체성이 생깁니다.

### 4.2 ncmod.json 필드

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "name": "My Mod",
  "description": "one-line description",
  "authors": ["someone"],
  "license": "MIT",
  "contact": { "homepage": "https://...", "sources": "https://..." },
  "icon": "icon.png",
  "environment": "both",
  "entry": "MyMod.ModEntry",
  "hooks": []
}
```

| 필드 | 필수 | 비고 |
| --- | --- | --- |
| `id` | 예 | 모드 식별자, 종속성과 조회에 사용됨. 빈 문자열은 스캐너가 건너뜀 |
| `version` | 권장 | 버전 번호; 다른 모드가 의존할 때 버전 제약이 이를 기준으로 판단됨, 아래 참고 |
| `name` | 아니오 | 표시 이름; 모드 페이지에 표시되는 것, 기본값은 `id` |
| `description` | 아니오 | 한 줄 설명 |
| `authors` / `contributors` | 아니오 | 크레딧, 문자열 배열 |
| `license` | 아니오 | 라이선스 식별자 |
| `contact` | 아니오 | 외부 링크; `homepage` / `sources` / `issues`를 받을 수 있음 |
| `icon` | 아니오 | 아이콘의 임베디드 리소스 이름, 4.7 참고 |
| `environment` | 아니오 | `both` / `client` / `server`, 기본값 `both`. 현재 측과 맞지 않으면 모드 전체가 로드되지 않음 |
| `entry` | 예 | 엔트리 클래스의 전체 이름; 클래스에 `public Task Init()`이 있어야 함 |
| `depends` | 아니오 | 의존하는 다른 모드와 요구 버전, 아래 참고 |
| `hooks` | 아니오 | 주입 규칙 목록; 빈 배열은 ModApi가 이미 가진 이벤트만 구독한다는 뜻 |
| `mixins` | 아니오 | 믹스인 규칙 목록; 이 모드의 클래스 중 하나의 멤버를 대상 타입으로 이동함, [2.8](#28-믹스인-대상-타입에-멤버-추가) 참고 |

`depends`는 의존하는 다른 모드와 요구 버전을 선언합니다; 키는 모드 id이고 값은 버전 제약입니다:

```json
{
  "depends": {
    "othermod": "^1.0.0",
    "another-mod": ">=2.1"
  }
}
```

**`depends`에 작성하지 않은 종속성에는 버전 제약이 없습니다**. 모드 간 종속성은 이미 컴파일 시점 참조에서 자동으로 추론되며([6.1](#61-모드-종속성--의존하는-모드-주입) 참고), `depends`는 그 위에 버전 제약만 추가합니다. 의존 대상 모드가 현재 측에 없으면 검사를 건너뜁니다 — 그 경우는 어셈블리 확인이 보고하도록 남겨 둡니다.

| 제약 문법 | 의미 |
| --- | --- |
| `*` | 모든 버전, 항목 생략과 동등 |
| `1.2.3` | 세그먼트 접두사; `1.2`는 `1.2`와 `1.2.9`에 매칭되지만 `1.3`에는 아님 |
| `^1.2.3` | 같은 메이저 버전이며 기준 이상; 메이저 버전이 0이면 마이너 버전을 대신 사용하므로 `0.1`과 `0.2`는 호환되지 않는 것으로 간주 |
| `>=1.2.3` | 기준 이상 |

버전 번호는 각 세그먼트의 선행 숫자만 취하므로 `26.2-netcraft`는 `26.2`로 참여합니다. 버전이 맞지 않으면 **선언한 모드만 건너뛰고** 나머지는 평소처럼 로드됩니다; 시작 로그에 "dependency X requires version …, actual version …"이 표시됩니다.

판단 기준은 의존 대상 모드 매니페스트의 `version` 필드입니다. 따라서 **의존 대상이 되려는 모드는 `version`을 올바르게 설정해야 합니다** — 빈 버전 번호는 어떤 구체적 제약도 충족하지 못합니다.

### 4.3 hook 규칙 필드

| 필드 | 비고 |
| --- | --- |
| `target` | 대상 타입의 전체 이름; 커널 어셈블리나 `mods/` 아래 모드 어셈블리 중 하나에 있어야 함 |
| `method` | 대상 메서드 이름; 동명 오버로드가 모두 매칭됨 |
| `type` | 주입 형태, 부록 참고 |
| `patchMode` | 적용 방식, `ILRewrite`(기본값) 또는 `RuntimeInject`, [2.6](#26-런타임-주입-이미-실행-중인-코드-수정) 참고 |
| `ordinal` | 같은 앵커가 호스트 메서드의 여러 곳과 매칭될 때 어느 것을 고를지, 0부터 시작, [2.4](#24-단일-지점으로-좁히기-호스트-스코핑과-배치) 참고 |
| `replaceType` | 대체 메서드를 포함하는 클래스의 전체 이름 |
| `replaceMethod` | 대체 메서드 이름 |
| `label` | 프로브 레이블, `Mark`와 `Probe`에서만 사용됨 |
| `environment` | 규칙이 적용되는 측, 기본값 `both`; 서버 타입을 대상으로 하는 규칙을 클라이언트에서 실행하면 대상이 전혀 없으므로 `environment`로 필터링됨 |

같은 규칙은 대체 메서드 위의 애노테이션으로도 작성할 수 있습니다; 대응은 다음과 같습니다:

```csharp
[Inject(typeof(SomeType), nameof(SomeType.SomeMethod),
        HookType = "CallSite", Label = "some_label", Environment = "server")]
public static void OnSomeMethod(object self) { }
```

| 매니페스트 필드 | 애노테이션 형태 |
| --- | --- |
| `target` | 첫 번째 생성자 매개변수, `typeof(...)`로 작성 |
| `method` | 두 번째 생성자 매개변수, 되도록 `nameof(...)` |
| `type` | 명명 매개변수 `HookType`, 기본값 `CallSite` |
| `patchMode` | 명명 매개변수 `PatchMode`, 기본값 `ILRewrite` |
| `ordinal` | 명명 매개변수 `Ordinal` |
| `label` | 명명 매개변수 `Label` |
| `environment` | 명명 매개변수 `Environment`, 기본값 `both` |
| `replaceType` | 작성하지 않음; 애노테이션이 붙은 클래스에서 자동으로 가져옴 |
| `replaceMethod` | 작성하지 않음; 애노테이션이 붙은 메서드에서 자동으로 가져옴 |

애노테이션은 `NetCraft.ModApi.Extension`의 `InjectAttribute`에서 오므로, 애노테이션을 사용하는 모드는 이를 참조해야 합니다. 애노테이션과 매니페스트가 둘 다 있으면 병합되며, 같은 주입 지점을 양쪽에 선언하면 **애노테이션이 이깁니다**.

믹스인 규칙은 다른 필드 집합을 사용합니다: `target`(대상 타입의 전체 이름), `source`(소스 타입의 전체 이름, 모드 자체 어셈블리에 있어야 함), `interfaces`(선택 사항, 인터페이스 전체 이름 배열)입니다; 의미론은 [2.8](#28-믹스인-대상-타입에-멤버-추가)을 참고하세요. 소스 클래스의 `[Mixin(typeof(target))]`는 매니페스트 항목과 동등합니다.

### 4.4 훅할 수 있는 것과 없는 것

훅할 수 있음: `kernel/` 아래의 커널 어셈블리, 그리고 `mods/` 아래의 다른 모드.

훅할 수 없음:

- 메인 라이브러리 `NetCraft.dll`
- 로더 `NetCraft.ModLoader.dll`
- 엔트리 어셈블리(`NetCraft.Server.Exe.dll` 등)

모든 훅의 `target`은 커널이나 어떤 모드 어셈블리에서 찾을 수 있어야 하며, 그렇지 않으면 구성 단계에서 "the injection target is not in any known assembly"를 보고합니다. **네임스페이스가 어셈블리를 함축하지 않는다**는 점에 주목하세요 — `NetCraft.Game.Server.DedicatedServer`는 실제로 `NetCraft.Server.dll`에 있습니다; 로더는 메타데이터 테이블로 만든 인덱스로 찾으므로 전체 이름만 작성하면 됩니다.

모드의 모드 주입도 같은 시스템을 따릅니다: 대상 모드는 `mods/`의 순서와 무관하게 **그 자체가 로드되는 순간** 재작성됩니다. 순환 규칙(A가 B를 주입하고 B가 A를 주입)은 로드 시 "cyclic loading"을 보고합니다; 이런 규칙은 IL 재작성 수준에서 해결책이 없으므로 하나를 제거하세요.

주입 대상 모드는 **자기 자신의 모드일 수 없습니다** — 자기 주입도 순환으로 간주됩니다.

### 4.5 배포

컴파일된 dll을 출력 디렉터리의 `mods/`(최상위, 하위 디렉터리 재귀 없음)에 복사하고 프로세스를 재시작하세요.

**이 단계는 빌드가 이미 자동화합니다**: `NetCraft.ModApi.csproj`의 `DeployModToHosts`가 빌드 후 dll을 각 호스트 프로젝트 출력 디렉터리의 `mods/` 디렉터리로 복사합니다. 호스트 목록은 `ModHostProjects` 속성이며, 자체 호스트 프로젝트를 만들 때 그 이름을 추가하면 됩니다. 규칙이 적용되지 않는데 오류도 전혀 없다면, 먼저 호스트 프로젝트의 `mods/`에 있는 dll이 오래된 것인지 확인하세요.

시작 로그는 다음과 같이 출력합니다:

```
Rewrote and preloaded assembly NetCraft.Server
Rewrote and preloaded assembly NetCraft.Game
Mod scan finished: 1 found, injection targets NetCraft.Server,NetCraft.Game runtime
Mod init finished: 1 loaded, 0 skipped
```

문제 해결 순서:

- 규칙이 적용되지 않음: 먼저 `mods/`의 dll이 오래된 것인지 확인하세요(가장 흔함).
- `Rewrote and preloaded assembly`가 보이지 않으면 대상 어셈블리가 부트스트랩 전에 이미 로드된 것이므로 규칙을 너무 늦게 작성한 것입니다.
- `injection targets`가 보이는데 대상이 잘못되었으면 보통 `target` 오타입니다; [4.4](#44-훅할-수-있는-것과-없는-것)에 따라 그 타입이 어느 어셈블리에 속하는지 확인하세요.
- `environment`가 `client`로 설정된 규칙은 서버에서 조용히 건너뜁니다(모드 수준의 불일치는 모드 전체가 로드되지 않는다는 뜻입니다); 이는 예상된 동작이며 결함이 아닙니다.

### 4.6 서드파티 종속성

Fabric의 Jar-in-Jar에 해당합니다.

템플릿으로 만든 프로젝트는 **이것을 걱정할 필요가 없습니다**: `dotnet add package`로 추가한 라이브러리는 빌드 시 자동으로 모드 dll에 임베드되며, 로더가 어셈블리를 확인할 수 없으면 모드의 임베디드 `.dll` 리소스를 살펴봅니다.

```
dotnet add package Newtonsoft.Json
```

그게 전부입니다; `ncmod.json`은 변경할 필요가 없습니다.

직접 작성한 프로젝트는 템플릿 csproj에서 `EmbedDependencies` 타깃을 복사하거나 직접 임베드해야 합니다:

```xml
<ItemGroup>
  <EmbeddedResource Include="deps\MyLib.dll" LogicalName="MyLib.dll" />
</ItemGroup>
```

종속성은 매니페스트에 작성하지 않습니다; 확인은 모드의 임베디드 `.dll` 리소스만 살펴보며, 리소스 이름이 어셈블리 이름과 일치하기만 하면 됩니다.

세 가지 참고:

- **매칭은 어셈블리 이름으로 합니다**. 로더는 요청된 어셈블리 이름을 비교합니다: `.dll`을 제거한 후 리소스 이름이 어셈블리 이름과 같거나 `.` + 어셈블리 이름으로 끝나면 됩니다. 따라서 `MyLib.dll`과 기본값 `MyProject.deps.MyLib.dll` 모두 작동합니다.
- **모드 자체의 임베디드 리소스만 인식됩니다**. 임베드되지 않은 라이브러리는 확인할 수 없으며 다른 곳에서 검색되지도 않습니다.
- **확인 순서는 커널 우선입니다**. `EmbeddedAssemblyLoader`는 커널 어셈블리와 임베디드 서브 라이브러리를 먼저 살펴보고, 아무것도 찾지 못한 경우에만 모드로 돌아가므로, 모드는 커널과 같은 이름의 어셈블리를 임베드하지 말아야 합니다.

템플릿의 타깃은 `Condition`을 작성하는 대신 `WithMetadataValue`로 `.dll`을 필터링합니다. 템플릿 엔진이 템플릿 시점에 `.csproj`의 `Condition`을 평가하는데, 그때 `%(...)`에 값이 없어 줄 전체가 삭제되기 때문입니다.

### 4.7 아이콘과 표시 정보

이름, 설명, 저자, 링크, 아이콘은 모두 `ncmod.json`에 작성하며, 바닐라 `fabric.mod.json`의 `name` / `description` / `authors` / `contact` / `icon`에 대응합니다.

**아이콘은 임베디드 리소스**이며 외부 파일이 아니고, 임베디드 종속성과 같은 리소스 명명 규칙을 사용합니다:

```xml
<ItemGroup>
  <EmbeddedResource Include="icon.png" LogicalName="icon.png" />
</ItemGroup>
```

```json
{ "icon": "icon.png" }
```

템플릿은 두 곳이 모두 설정된 `icon.png`를 이미 제공하므로 그 이미지만 교체하면 됩니다. 64×64 또는 128×128 PNG를 권장합니다.

아이콘 조회 순서는: 매니페스트의 `icon`이 가리키는 것 → `icon.png`라는 이름의 임베디드 리소스 → 둘 다 없으면 UI의 기본 이미지(회색 물음표)입니다.

이 정보는 서버 GUI의 **MODS** 페이지에서 볼 수 있습니다. 왼쪽은 모드 목록(작은 아이콘 + 표시 이름 + 버전 + 상태)이고; 오른쪽은 선택한 모드의 설명, 크레딧, 링크, 종속성, 초기화 시간, 주입 규칙입니다. 종속성 열의 모드 이름은 클릭할 수 있고 해당 항목으로 바로 이동합니다.

로드에 실패했거나 건너뛴 모드도 목록에 있으며 상태 열에 표시됩니다 — "내 모드가 왜 적용되지 않았지"를 진단할 때 이 열을 먼저 확인하세요.

---

## 5. 알려진 제한 사항

- **프로브 시그니처는 BCL 타입과 `object`만 사용할 수 있습니다**, 이유는 2.2에 있습니다; 값 타입 매개변수와 반환값은 예외이며 실제 타입을 유지해야 합니다.
- **`CallSite`는 교체입니다**, 그리고 대체 메서드가 원본 호출을 직접 복원해야 합니다(2.3 참고). private 메서드는 복원할 수 없어 리플렉션이 필요합니다.
- **`mods/` 아래의 dll은 `DeployModToHosts`가 자동으로 배포합니다**; 자체 호스트 프로젝트를 만들 때 `ModHostProjects`에 그 이름을 추가하는 것을 잊지 마세요.
- **엔트리 어셈블리에는 주입할 수 없습니다**: 대상과 훅이 엔트리 어셈블리에 속하면 효과가 없습니다.
- **메인 라이브러리와 로더에는 훅할 수 없습니다**; 모드가 로딩 과정 자체를 바꾸지 못하게 하는 설계상의 제약입니다.
- **`environment`가 맞지 않으면 모드 전체가 로드되지 않습니다**, "일부 규칙 실패"가 아닙니다.
- **런타임 주입에는 네이티브 라이브러리가 필요합니다**: `RuntimeInject`를 사용하는 규칙은 프로세스 시작 시 `lead_hook_native`가 부착되어야 하며, 로더가 이를 위해 스스로 재시작합니다; 라이브러리를 찾지 못하거나 재시작이 실패하면 이 규칙 묶음은 경고로 강등되고 시작은 막히지 않습니다. 기능 경계와 비용은 [2.6](#26-런타임-주입-이미-실행-중인-코드-수정)을 참고하세요.
- **애노테이션은 C# API보다 필드가 적습니다**: `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue`는 `[Inject]`에 쓸 수 없습니다(`PatchMode`와 `Ordinal`은 지원됨), [2.1](#21-주입-방식-애노테이션-또는-매니페스트-하나를-고르세요) 참고.
- **믹스인은 로드 시 재작성에만 적용됩니다**, 소스 클래스의 멤버는 복사가 아니라 이동되며; 소스 클래스의 중첩 타입과 제네릭 메서드는 범위 밖이고, 인터페이스와 함께 믹스인된 메서드는 virtual로 표시됩니다. [2.8](#28-믹스인-대상-타입에-멤버-추가) 참고.
- 테스트와 디버깅 시 언어 테이블이나 모델 리소스를 사용하면 (바닐라 jar에서 추출한) `assets` 디렉터리가 필요하며, 그렇지 않으면 관련 기능이 번역 키나 플레이스홀더 텍스처로 강등됩니다.

---

## 6. 알려진 동작

이 장은 관찰된 동작의 기록이며 명세가 아닙니다.

### 6.1 모드 종속성 + 의존하는 모드 주입

**시나리오**: b가 a에 의존하고 a도 주입합니다.

**결론**: 작동하며 순환을 형성하지 않습니다.

체인은 세 단계입니다:

1. 규칙 테이블은 어셈블리를 전혀 로드하지 않고 PE 메타데이터를 읽어서만 만들어집니다. "b가 a를 주입"이라는 규칙은 a도 b도 존재할 필요가 없습니다.
2. `PreloadReplacers`가 `ModManager` 전에 모든 대체 클래스(b 포함)를 로드하며, 이 시점에 a는 아직 로드되지 않았습니다. `LoadFromStream`은 메타데이터만 읽고 메서드 본문을 JIT하지 않으므로, 이 순간 b의 a에 대한 참조는 지연 상태이고 로딩이 실패하지 않습니다.
3. 그런 다음 `ModManager`가 `AssemblyRef`로 위상 정렬하며 a가 b보다 앞입니다. a를 재작성할 때 대체 클래스 b는 이미 `Default`에 있으므로 타입을 직접 가져오고, b를 가리키는 `AssemblyRef` 하나가 a의 메타데이터에 추가됩니다. **a는 b가 존재한다는 것을 전혀 알 필요가 없습니다.**

**엄격한 요구 사항**: csproj에서 주입 대상 모드를 참조할 때 `Private="false"`를 반드시 작성해야 합니다. 기본값으로는 `a.dll`이 출력 디렉터리로 복사되고, 그러면 `EmbedDependencies`가 이를 임베디드 종속성으로 `b.dll`에 임베드하며, 런타임에 `ModLibs`가 `a`를 확인할 때 두 번째 복사본을 집어 들어 둘 사이의 타입 정체성 검사가 실패합니다.

**검증 사례**: 두 템플릿 프로젝트 `NetCraft.Test1`(a)과 `NetCraft.Test2`(b)입니다; a는 `Test1Api.Greet`을 제공하고 자체 `ModEntry.Server`에서 호출하며, b는 그 호출 지점을 `Test2Probe.OnGreet`으로 교체하고 대조군으로 b의 `ModEntry.Server`에서도 `Greet`을 한 번 호출합니다. 실제 실행 로그:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]
Mod scan finished: 2 found, injection targets NetCraft.Server,NetCraft.Game,NetCraft.Storage,NetCraft.Test1 runtime
Test1 internal greeting: Test2-rewritten greeting self    ← 주입이 적용됨
Test2 internal greeting: Test1 original greeting test2    ← 대조군: 규칙은 대상 어셈블리만 재작성하므로 b 자체의 내부 호출 지점은 건드려지지 않음
```

종속성은 `AssemblyRef`에서만 순서를 도출하며 버전 제약을 담지 않습니다. 버전을 제한하려면 매니페스트의 `depends`에 선언하세요, [4.2](#42-ncmodjson-필드) 참고.

### 6.2 애노테이션 주입

**시나리오**: 대체 메서드에 `[Inject(typeof(X), nameof(X.M))]`가 있고 `ncmod.json`에는 규칙이 없습니다.

**결론**: 작동하며 매니페스트 선언이 필요 없습니다. 애노테이션과 매니페스트는 하나의 소스를 공유하고 어셈블리 구성 시 병합되며, 같은 주입 지점에는 애노테이션이 이깁니다.

**왜 선언이 필요 없는가**: 애노테이션은 메타데이터의 `CustomAttribute` 테이블일 뿐이고, 로더는 모드를 스캔할 때 이미 같은 메타데이터(매니페스트, 임베디드 리소스, AssemblyRef — 세 항목)를 읽으므로, 테이블 하나를 더 읽는다고 새로운 로딩이나 타이밍 제약이 생기지 않습니다. 유일한 전제 조건은 모드가 `NetCraft.ModApi`(애노테이션의 호스트)를 참조하는 것입니다.

**전제는 정적 읽기입니다**: 애노테이션 읽기는 `MetadataReader`를 거쳐야 하며 **`Assembly.Load` + `GetCustomAttributes`를 사용해서는 안 됩니다** — 후자는 규칙을 읽기만 하려고 모드 어셈블리를 끌어올려 재작성 창이 즉시 사라집니다.

**`typeof`는 타입 참조를 구성하지 않습니다**: `typeof(X)`가 매개변수로 컴파일하는 것은 타입의 직렬화된 이름(`full name, assembly, Version=…`)이며, 이는 문자열로 해석될 뿐 `X`가 존재할 필요가 없습니다. 따라서 애노테이션에 `typeof(injected-mod)`를 작성해도 2.2의 "대체 클래스는 주입 대상 모드의 타입을 참조해서는 안 된다"는 제약을 **위반하지 않습니다** — 메타데이터에 이름이 나타나는 것과 런타임에 타입을 해석하는 것은 다른 일입니다.

**검증 사례**: `NetCraft.Test2`가 `NetCraft.Test1`의 두 메서드를 대상으로 합니다; `Greet`은 애노테이션을, `Farewell`은 매니페스트를 거칩니다. 두 규칙 모두 설치되고 두 호출 지점 모두 교체됩니다:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]        ← 애노테이션
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Farewell replaced by NetCraft.Test2.Test2Probe::OnFarewell [CallSite/ILRewrite]  ← 매니페스트
Test1 internal greeting: Test2-rewritten greeting self
Test1 internal farewell: Test2-rewritten farewell self
Test2 internal greeting: Test1 original greeting test2     ← 대조군: 규칙은 대상 어셈블리만 재작성함
Test2 internal farewell: Test1 original farewell test2     ← 대조군
```

같은 사례는 애노테이션의 명명 매개변수(`Environment = "server"`)도 해석된다는 것을 덤으로 검증했습니다.

### 6.3 프로파일러 부착의 성능 비용

**시나리오**: 같은 순수 계산 프로그램(1억 회 나머지-누적 반복)을 프로파일러 없이 한 번, 부착한 상태(세 개의 `CORECLR_ENABLE_PROFILING` 환경 변수)로 한 번 실행합니다. 각각 다섯 번.

**결론**: 정상 상태에서는 차이가 없고, 비용은 전적으로 시작 시점에 있습니다.

| | 총 프로세스 시간(5회, ms) | 프로그램 내 계산 시간 |
| --- | --- | --- |
| 없이 | 306 / 260 / 278 / 253 / 300 | 225 ms |
| 부착 시 | 406 / 370 / 340 / 325 / 354 | 193 ms |

총 시간이 약 80~110ms 더 깁니다. 원인은 `COR_PRF_DISABLE_ALL_NGEN_IMAGES`입니다 — ReJIT를 활성화하려면 ReadyToRun 이미지를 동시에 비활성화해야 하므로 프레임워크 코드는 JIT를 거칠 수밖에 없고; 프로그램 내부의 타이밍 루프는 JIT 컴파일되면 동일하여 차이가 나타나지 않습니다(프로파일러를 부착한 실행이 실제로 약간 더 빨랐는데 이는 노이즈입니다).

**덤으로 고친 낭비**: 초기 구현은 `COR_PRF_MONITOR_JIT_COMPILATION`을 구독했는데, 이는 모든 메서드가 컴파일된 후 발생하는 네이티브 콜백으로 우리가 전혀 사용하지 않았습니다. 이를 제거한 후 이벤트 마스크가 `0x80040024`에서 `0x80040004`로 바뀌었으며, 위 표는 제거 후의 데이터입니다.

**검증 사례**: `__hookverify/BenchProbe`.

### 6.4 같은 대상을 주입하는 두 모드

**시나리오**: 두 모드가 각각 같은 대상 메서드에 걸리는 규칙을 선언합니다(`MethodBody` 형태, 서로 다른 대체 메서드).

**결론**: 오류도 충돌도 없습니다; **먼저 구성된 규칙이 이기고 나중 것이 조용히 실패합니다**.

두 규칙 모두 규칙 테이블에 들어갑니다 — 모드 간 중복 제거는 하지 않습니다. 재작성 시 `OriginalType::OriginalMethod`의 같은 키 목록에서 **첫 번째** 항목을 취합니다; 호스트 스코핑도 같은 방식이며 `InType`/`InMethod`는 "첫 매칭이 이김"입니다. 어느 것이 먼저인지는 구성 순서에 달려 있고, 구성 순서는 mods 디렉터리의 열거 순서에서 옵니다; **우선순위 필드가 없고 종속성 선언으로 제어할 수 없습니다**(종속성은 `Init()`의 순서에만 영향을 주며 주입 규칙의 구성에는 영향을 주지 않습니다).

**런타임 주입도 선착순입니다**: 나중 등록 요청은 정상적으로 전송되지만 `GetReJITParameters`가 "모듈 + 메서드"로 차지하며 항상 첫 요청과 매칭되므로 나중 등록은 적용되지 않습니다. 두 번째 주입 후 측정해도 대상 메서드의 동작은 첫 결과에 머뭅니다.

**하나의 잘못된 규칙은 다른 규칙에 영향을 주지 않습니다**: 대상 타입이 알려진 어셈블리에 없거나 주입 형태가 오타인 문제는 구성 시점에 `ModHooks.Errors`에 기록되고 그 항목은 건너뛰며, 다른 모드의 규칙은 평소처럼 구성됩니다.

**충돌은 기록됩니다**: 구성 단계에서 같은 주입 지점이 여러 모드에 선언된 것을 감지하면 나중에 구성된 쪽을 `ModHooks.Warnings`에 쓰고 시작 로그에 `Mod injection conflict ...`로 출력하며, 어떤 두 모드가 충돌했고 어느 것이 적용되지 않을지를 명시합니다. 오류가 아니라 경고이며 로딩에 영향을 주지 않고, 규칙 자체는 테이블에 남아 있습니다(도달할 수 없을 뿐).

**런타임 주입도 이 검사를 거칩니다**: 충돌 감지 키에 `patchMode`가 포함되므로 한 모드가 `ILRewrite`를, 다른 모드가 `RuntimeInject`를 쓰면 충돌로 간주되지 않습니다(독립된 두 경로가 각자 하는 일); 같은 모드 두 개만 충돌로 판정되어 경고됩니다. 실제 적용도 마찬가지로 선착순입니다 — `GetReJITParameters`가 "모듈 + 메서드"로 차지하며 첫 요청과 매칭되므로 나중 것은 전송되지만 적용되지 않습니다.

**유일한 강경 크래시 지점**: 두 모드가 같은 메서드를 `RuntimePatch` 모드로 모두 패치하면 두 번째가 `RuntimeHookEngine`의 중복 등록 검사에 걸려 `InvalidOperationException`을 던지고, 이 경로는 잡히지 않아 시작이 아예 실패합니다. `RuntimePatch`는 폐지 수순입니다([2.6](#26-런타임-주입-이미-실행-중인-코드-수정) 참고); 새 규칙에서는 사용하지 마세요.

**검증 사례**: `NetCraft.Test`의 `modinjection` 모듈, 항목 `same anchor first mod wins quietly`와 `one bad rule does not sink the rest`.

### 6.5 아직 검증되지 않음

- **실제 서버에서 런타임 주입이 작동하는지**: `RuntimeInject` 모드는 `__hookverify/RuntimeProbe`에서 종단 간 검증되었고(등록 후 대상 메서드의 동작이 대체 메서드의 것으로 바뀜), 모드 어셈블리 쪽도 라우팅과 강등에 대한 테스트 커버리지가 있습니다; 하지만 현재 어떤 빌드 단계도 `lead_hook_native`를 NC의 실행 디렉터리에 넣지 않으므로, 실제 서버에서 이 체인을 실행하려면 먼저 라이브러리를 프로그램 루트에 두어야 합니다(또는 `NC_PROFILER_PATH`로 가리켜야 합니다). 이 단계는 아직 하지 않았습니다.
- **주입 대상 모드의 타입을 참조하는 대체 클래스**: 추론상 a를 재작성하는 순간 a를 해석하려 하지만, a는 로드 완료 직전에 멈춰 있습니다(아직 `Default`에 없고 `ModLibs`도 커널 확인 콜백도 모드 어셈블리를 인식하지 못함), 따라서 `PrepareMod`가 예외를 던지고 `result.Errors`에 기록될 것으로 예상됩니다. 아직 실제로 실행하지 않았습니다. 6.2는 **애노테이션의 `typeof`**가 참조가 아님을 증명할 뿐이며, **메서드 시그니처에 타입이 나타나는 것**은 별개입니다.

---

## 부록: HookType 개요

| 형태 | 효과 | 대체 메서드 시그니처 요구 사항 |
| --- | --- | --- |
| `CallSite` | 대상 메서드에 대한 호출 지점을 여러분의 메서드로 교체 | 매개변수 수가 호출된 메서드와 일치(인스턴스 호출 +1) |
| `MethodBody` | 대상 메서드 본문 전체를 교체 | 교체된 메서드와 일치 |
| `NewObj` | `new X(...)`를 교체 | 매개변수 수가 생성자와 일치 |
| `FieldRead` | 필드 읽기를 계측 | 읽기 타입에 따름 |
| `FieldWrite` | 필드 쓰기를 계측 | 쓰기 타입에 따름 |
| `TypeCheck` | `isinst` / `castclass`를 계측 | 검사되는 타입에 따름 |
| `Box` | 박싱/언박싱을 계측 | 요소 타입에 따름 |
| `FunctionPointer` | 함수 포인터 로드를 계측 | 델리게이트 타입에 따름 |
| `LocalRead` | 지역 변수 읽기를 계측 | 매개변수 없음, 변수 값을 반환 |
| `LocalWrite` | 지역 변수 쓰기를 계측 | 매개변수 하나, 쓰인 값을 받음 |
| `Constant` | 상수 로드를 계측 | 매개변수 없음, 상수 값을 반환 |
| `Probe` | 원본 메서드 본문을 유지하고 진입과 모든 종료를 계측; `LabelArgumentIndex`를 사용하면 인수 하나를 레이블에 접을 수 있음 | `Begin()`은 long 반환, `End(string, long)` |
| `Mark` | 메서드 진입 시 한 번만 보고, 타이밍 없음 | `void method(string label)` |

`Probe`와 `Mark`는 레이블 텍스트만 전달합니다(`Probe`는 인수 하나의 `ToString()`도 포함할 수 있음); 객체 참조를 얻을 수는 없습니다. 실제 인수를 얻으려면 `CallSite`를 사용하세요.

`InType`/`InMethod`, `Placement`, `Ordinal`([2.4](#24-단일-지점으로-좁히기-호스트-스코핑과-배치) 참고)은 명령 수준 형태에만 의미가 있습니다: 위 표에서 `MethodBody`, `Probe`, `Mark`를 제외한 열 개 항목은 교체 또는 앞/뒤 삽입을 선택할 수 있고 `Ordinal`로 한 번의 발생을 고를 수 있습니다; `MethodBody`는 항상 전체를 교체하고 `Probe`/`Mark`는 이 매개변수를 무시합니다.

세 종류 `LocalRead` / `LocalWrite` / `Constant`의 경우, 호스트 메서드는 참조된 엔티티가 아니라 `target`에 작성하며, 추가로 `localIndex` 또는 `constantValue`가 필요합니다; [2.5](#25-메서드-본문-내-앵커-지역-변수와-상수) 참고.
