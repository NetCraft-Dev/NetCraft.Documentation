# NetCraft-ModApi 참조

`NetCraft.ModApi`은 NetCraft가 모드에 노출되는 API 표면입니다. 여기에는 두 가지 정체성이 있습니다. 여러분에게는 API 라이브러리입니다. 그 자체로는 일반 모드입니다(`id`은 `netcraft-modapi`이며 자체 `ncmod.json` 및 주입 프로브를 제공합니다).

이 파일은 API가 커짐에 따라 커집니다. 건축 배경, Fabric과의 차이점, 모드 작성 방법은 [modding-guide.md](modding-guide.md)를 참조하세요.

- 조립 : `NetCraft.ModApi.dll`

- 종속성: `NetCraft`(메인 라이브러리), `NetCraft.Game`

공개 표면은 세 개의 네임스페이스로 분할됩니다.

| 네임스페이스 | 내용 | 메모 |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | 이벤트 및 구독 기본 클래스 `NcEvent<T>`, `Nc*` 외관, `Nc*` 객체 핸들 | 래퍼 레이어; 공개 표면에 커널 유형이 없습니다 |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 주석 | 확장점; 규칙은 커널 클래스 및 메소드 이름에 바인딩 |
| `NetCraft.ModApi.Internal` | 주입 프로브 | 직접 참조하지 마세요 |

루트 네임스페이스 `NetCraft.ModApi`에는 항목 클래스 `ModApiEntry`만 포함되어 있습니다. `Wrapper`와 `Extension`는 두 개의 병렬 경로입니다. 선택 방법은 [modding-guide.md 2.9](modding-guide.md#29-two-routes-wrapper-layer-and-extension-points)를 참조하세요.

***

## 1. 빠른 시작

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //subscription returns a handle; disposing it unregisters
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //you can also do something else with the handle
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

`ncmod.json`에서 이 클래스를 `entry`로 지정하고 `hooks`를 비워 둡니다. 아래 이벤트는 모두 ModApi의 자체 프로브에서 제공됩니다.

***

## 2. 이벤트

모든 이벤트는 `NetCraft.ModApi.Wrapper` 아래에 있습니다. `using NetCraft.ModApi.Wrapper;` 이후에는 사용할 수 있습니다.

### 2.1 요약표

| 이벤트 | 인수 유형 | 트리거 | 측면 | ModApi 후크 포인트 |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick` | `ServerTickArgs` | 서버 메인 루프의 모든 틱 | 서버 | `DedicatedServer::Tick` (마크) |
| `ServerEvents.Started` | `ServerPhaseArgs` | `Done (x.xxxs)!`가 인쇄된 후 메인 루프가 시작되었습니다 | 서버 | `MinecraftServer::Run` (마크) |
| `ServerEvents.Stopping` | `ServerPhaseArgs` | 서버가 종료되기 시작합니다. 플레이어의 연결이 곧 끊어집니다 | 서버 | `DedicatedServer::Stop` (마크) |
| `ServerEvents.CommandRegister` | `CommandRegisterArgs` | 모든 내장 명령이 등록되었습니다 | 서버 | `EffectCommand::Register` 호출 사이트(CallSite) |
| `ServerEvents.PlayerJoin` | `PlayerJoinArgs` | 조인 패킷 시퀀스가 ​​전송되었습니다 | 서버 | `PlayerList::PlaceNewPlayer` 호출 사이트(CallSite) |
| `ServerEvents.PlayerLeave` | `PlayerLeaveArgs` | 플레이어가 온라인 목록에서 제거되었습니다 | 서버 | `PlayerList::RemovePlayer` 호출 사이트(CallSite) |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | 전송된 패킷 연결 끊기 및 연결 종료 | 서버 | `ServerPlayer::Disconnect` 호출 사이트(CallSite) |
| `ServerEvents.PlayerHurt` | `PlayerHurtArgs` | 실제로 입힌 피해 | 서버 | `PlayerList::HurtPlayer` 호출 사이트(CallSite) |
| `ServerEvents.PlayerDeath` | `PlayerDeathArgs` | 체력이 0으로 재설정된 직후 | 서버 | `PlayerList::RespawnPlayer` 호출 사이트(CallSite) |
| `ServerEvents.PlayerChat` | `PlayerChatArgs` | 채팅 방송 후 | 서버 | `ServerGamePacketListenerImpl::HandleChat` 호출 사이트(CallSite) |
| `ServerEvents.ChunkLoaded` | `ChunkLoadedArgs` | 청크가 처음으로 메모리에 들어갑니다 | 서버 | `ServerChunkCache::set_ChunkLoaded` 할당 사이트(CallSite) |
| `ServerEvents.ChunkUnloaded` | `ChunkUnloadedArgs` | 덩어리는 기억을 떠난다 | 서버 | `ServerChunkCache::set_ChunkUnloaded` 할당 사이트(CallSite) |
| `ServerEvents.ChunkSaved` | `ChunkSavedArgs` | 청크가 디스크에 기록되기 전에 찍은 스냅샷 | 서버 | `ServerChunkCache::set_ChunkSaveSink` 할당 사이트(CallSite) |
| `ServerEvents.CommandExecuted` | `CommandExecutedArgs` | 명령 실행이 완료되었습니다. 구문 오류 및 권한 거부도 포함됩니다 | 서버 | `CommandManager::Execute`(콜 사이트) |
| `ServerEvents.LevelTick` | `LevelTickArgs` | 레벨 틱, 틱당 로드된 레벨당 한 번 | 서버 | `PersistentServerLevel::Tick`(콜 사이트) |
| `ServerEvents.SavedDataSaving` | `SavedDataSavingArgs` | 청크 저장보다 한 단계 늦은 디스크에 저장된 데이터 | 서버 | `SavedDataStorage::ScheduleSave`(콜 사이트) |
| `ServerEvents.BlockChanged` | `BlockChangedArgs` | 블록 상태가 변경되어 클라이언트와 동기화하려고 합니다 | 서버 | `IBlockUpdateSink::BlockChanged`(콜 사이트) |
| `ServerEvents.BlockBroken` | `BlockBrokenArgs` | 블록이 깨졌습니다. 플레이어 채굴 및 레드스톤 자멸 모두 포함 | 서버 | `ServerBlockUpdates::BreakBlock`(콜 사이트) |
| `ServerEvents.ItemDropped` | `ItemDroppedArgs` | 블록 브레이크 드롭 및 요리 제품을 포함하여 드롭된 아이템 개체가 생성됨 | 서버 | `ServerBlockUpdates::SpawnDrop`(콜 사이트) |
| `NetworkEvents.PacketReceived` | `PacketReceivedArgs` | 핸드셰이크 및 상태 단계를 포함하여 핸들러에 대기 중인 모든 인바운드 패킷 | 둘 다 | `PacketProcessor::ScheduleIfPossible` 및 `HandleNow`(CallSite) |
| `ClientEvents.Tick` | `ClientTickArgs` | 클라이언트 메인 루프의 모든 틱 | 클라이언트 | `MinecraftClient::Tick` (마크) |

플레이어 이벤트 순서: 죽음은 상처 흐름 내부에 중첩되어 있으므로 `PlayerDeath`이 해당 `PlayerHurt` 앞에 옵니다. `PlayerLeave`와 `PlayerDisconnect`는 서로 다른 두 가지입니다. 전자는 온라인 목록에서 제거를 의미하며(`/kick` 이후에도 연결이 끊어진 후에만 실행됨), 후자는 연결 자체가 끊어짐을 의미하며 둘이 쌍으로 표시된다는 보장은 없습니다.

### 2.2 구독 및 등록 취소

```csharp
IDisposable Subscribe(Action<T> handler)
```

- 동일한 이벤트를 여러 번 구독하면 구독 순서대로 알림이 전달됩니다.

- 디스패치는 콜백 목록의 스냅샷을 찍기 때문에 콜백 내부에서 구독하거나 등록을 취소해도 현재 디스패치에는 영향을 미치지 않습니다.

- 등록을 취소하지 않아도 영구적으로 유효합니다. 모드는 언로드 메커니즘을 제공하지 않으므로 일반적으로 수동으로 등록을 취소할 필요가 없습니다.

### 2.3 인수 유형

**`ServerTickArgs`**

| 부동산 | 유형 | 메모 |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | 이번 출시 이후 틱 수(1부터 시작) |

이는 커널의 `TickCount`이 아니라 ModApi 자체에 의해 계산됩니다.

**`ClientTickArgs`**

| 부동산 | 유형 | 메모 |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | 위와 동일, 클라이언트 측에서 독립적으로 계산됨 |

**`ServerPhaseArgs`**

| 부동산 | 유형 | 메모 |
| -------- | -------- | ---------------------------- |
| `Phase` | `string` | 상 이름, `started` 또는 `stopping` |

필드는 이벤트 자체를 복제합니다. 로깅이 하나의 통일된 형식을 사용할 수 있도록 유지됩니다.

**`CommandRegisterArgs`**

| 회원 | 유형 | 메모 |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher` | `CommandDispatcher<CommandSourceStack>` | 커널의 명령 디스패처 |
| `Register(name, description, build)` | 방법 | 명령을 등록하고 원장에 기록합니다. 3.1 |

**`PlayerJoinArgs`**

| 부동산 | 유형 | 메모 |
| ------------- | -------------- | ---------------------------------------------- |
| `Player` | `ServerPlayer` | 방금 합류한 플레이어; 이미 전송된 조인 패킷, 상태를 읽어도 안전함 |
| `ProfileName` | `string` | 플레이어 이름 |

**`PlayerLeaveArgs`**

| 부동산 | 유형 | 메모 |
| --------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | 이 시점에서 더 이상 온라인 목록에 없는 떠나는 플레이어 |
| `Removed` | `bool` | 실제로 제거되었는지 여부; `false` 반복 제거 시 |

**`PlayerDisconnectArgs`**

| 부동산 | 유형 | 메모 |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | 연결이 끊긴 플레이어 |
| `Reason` | `string` | 연결 끊김 이유; 구성 요소로 제공되는 경우 일반 텍스트 |

연결 해제 패킷이 전송되었으며 연결이 닫혔습니다. 이제 이 플레이어에 패킷을 보내는 것은 효과가 없습니다.

**`PlayerHurtArgs`**

| 부동산 | 유형 | 메모 |
| ---------- | --------------- | -------------------------------------------- |
| `Player` | `ServerPlayer` | 부상당한 선수 |
| `Attacker` | `ServerPlayer?` | 피해를 입힌 플레이어; 환경 및 명령 피해에 대한 `null` |
| `Amount` | `float` | 이번 피해액 |

무적 프레임 동안이나 사망 후에는 실행되지 않습니다(커널의 `Hurt`가 `false`을 반환함).

**`PlayerDeathArgs`**

| 부동산 | 유형 | 메모 |
| ---------- | --------------- | ------------------------------ |
| `Player` | `ServerPlayer` | 죽은 플레이어 |
| `Attacker` | `ServerPlayer?` | 살인자; `null` 없는 경우 |

커널은 체력이 0에 도달한 후 즉시 재설정되므로 이벤트가 발생하면 플레이어는 리스폰 지점에서 이미 최대 체력을 유지하고 있습니다. 사망 순간의 좌표와 드롭은 사용할 수 없습니다.

**`PlayerChatArgs`**

| 부동산 | 유형 | 메모 |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | 보낸 사람 이름 |
| `Message` | `string` | 일반 텍스트 메시지 |

이 이벤트는 **읽기 전용 알림**입니다. 원래 메서드가 이미 메시지를 브로드캐스트했으므로 여기에서 `Message`을 변경해도 아무런 효과가 없습니다.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

세 가지 이벤트는 동일한 인수 형태를 공유합니다.

| 부동산 | 유형 | 메모 |
| -------- | ----- | ------------------ |
| `X` | `int` | 청크 좌표 X |
| `Z` | `int` | 청크 좌표 Z |

가장 오해하기 쉬운 것은 `ChunkSaved`입니다. 커널은 **동기적으로 스냅샷을 찍기** 위해 이 콜백을 요구하는 반면 직렬화와 디스크 쓰기는 커널 자체에 의해 비동기적으로 수행됩니다. 따라서 이 콜백에서 시간이 많이 걸리는 작업으로 인해 청크 언로드 속도가 직접적으로 느려지고 파괴적인 작업(블록 제거, 인벤토리 변경)도 여기에 있어서는 안 됩니다. 이는 스냅샷 순간만 약속할 뿐입니다.

`ChunkUnloaded`이 실행되면 블록 엔터티는 이미 청크와 함께 정리되었습니다. 블록을 읽으려면 `ChunkSaved`(블록에 액세스할 수 없음) 또는 이전 지점을 사용하십시오.

**`CommandExecutedArgs`**

| 부동산 | 유형 | 메모 |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string` | 원시 명령 텍스트; 채팅 명령에는 앞에 슬래시가 없습니다 |
| `Result` | `int` | 명령 반환 값; 0은 실패 또는 거부를 의미 |
| `Source` | `CommandSourceStack?` | 명령 소스; 플레이어 오버로드 경로의 `null` |
| `Player` | `ServerPlayer?` | 명령을 내린 플레이어; `null` 콘솔에서 실행된 경우 |

해당 이벤트는 명령이 완료된 **후** 발생합니다. 실행을 변경할 수 없습니다. 구문 오류 및 권한 거부도 여기에서 발생합니다. `Result`을 사용하여 구분하세요. 플레이어가 채팅 바에서 보내는 명령은 `Execute(ServerPlayer, string)` 오버로드를 거치며, 여기서 커널은 명령 소스를 내부적으로 구축하므로 이 경우 `Source`는 `null`이고 `Player`만 설정됩니다.

**`LevelTickArgs`**

| 부동산 | 유형 | 메모 |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level` | `NcLevel` | 이 틱의 레벨이 향상되었습니다 |
| `RunsNormally` | `bool` | 정상적으로 진행되었는지; `/tick freeze` 중 `false` |

틱당 로드된 레벨당 한 번 실행되므로 다중 레벨 세계는 틱당 여러 개를 수신합니다. 트리거 포인트는 레벨 틱 **완료** 이후입니다. 차단 지점이 아니라 관찰 지점입니다.

**`SavedDataSavingArgs`**

| 부동산 | 유형 | 메모 |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage` | 지속되는 저장된 데이터 테이블 |

이는 `ServerEvents.ChunkSaved`과 다른 경로입니다. 청크 1은 스냅샷을 만들고 비동기적으로 쓰기만 하는 반면, 이 청크는 동기 쓰기가 완료된 후에 실행됩니다. 세계 시계, 게임 규칙, 세계 경계 데이터가 이를 통과합니다.

**`BlockChangedArgs`**

| 부동산 | 유형 | 메모 |
| -------- | ------------ | --------------------------- |
| `Pos` | `BlockPos` | 변경된 블록의 위치 |
| `State` | `BlockState` | 변경 후 블록 상태 |

상태는 이미 청크에 기록되었으며 클라이언트와 동기화하려고 하므로 여기서 변경 사항 자체를 수정할 수 없습니다. 동작 콜백에서 자체 상태를 변경하는 Redstone 구성 요소도 이 경로를 통해 종료됩니다. 빈도가 높으므로 콜백에서 시간이 많이 걸리는 작업을 수행하지 마십시오.

**`BlockBrokenArgs`**

| 부동산 | 유형 | 메모 |
| -------- | --------------- | ------------------------------------------ |
| `Pos` | `BlockPos` | 깨진 블록의 위치 |
| `Player` | `ServerPlayer?` | 차단기; `null` 레드스톤과 같은 플레이어가 아닌 원인에 대해 |

블록이 실제로 교체될 때만 발생합니다. 빈 위치와 거부된 휴식 시간은 이를 트리거하지 않습니다. 브레이크 효과와 드롭은 이미 처리되었으므로 이벤트에서 읽는 내용이 결과입니다.

**`ItemDroppedArgs`**

| 부동산 | 유형 | 메모 |
| -------- | ----------- | ----------------------- |
| `Pos` | `BlockPos` | 떨어진 아이템이 나타난 곳 |
| `Stack` | `ItemStack` | 드롭된 아이템 스택 |

블록 브레이크 드롭과 캠프파이어 요리 제품 모두 여기를 통과합니다. 빈 항목 스택은 개체를 생성하지 않으므로 이벤트가 없습니다.

**`PacketReceivedArgs`**

| 부동산 | 유형 | 메모 |
| --------------- | -------- | ------------------------ |
| `Listener` | `object` | 이 패킷을 수신하는 수신기 |
| `Packet` | `object` | 패킷 객체 자체 |
| `IsServerbound` | `bool` | 서버바운드 패킷인지 여부 |

핸드셰이크, 상태, 구성 및 재생의 4단계를 모두 포함하는 모든 인바운드 패킷에 대해 실행됩니다. 이동 패킷은 틱당 여러 번 도착하므로 콜백에서 시간이 많이 걸리는 작업을 수행하지 마십시오. 패킷이 이미 객체로 디코딩되었지만 비즈니스 계층에 들어가지 않았습니다. 유형을 구별하려면 `Packet`을 직접 검사하세요. 아웃바운드 패킷은 이 이벤트 범위를 벗어납니다.

***

## 3. 확장 지점

### 3.1 명령어 등록

타이밍은 `ServerEvents.CommandRegister`입니다. 이 이벤트의 인수를 캐시하지 마십시오. 내부 명령 트리는 시작 시 한 번만 빌드됩니다.

```csharp
public void Register(
    string name,                                          //command literal, without the slash
    string description,                                   //one-line description, shown in the /ncmapi ledger
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //attach arguments and the executor
```

당신이 받는 `build` 은 커널의 준장 빌더입니다; 인수, 하위 명령 및 권한을 작성하면 커널의 방식이 결정됩니다.

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //permission predicate
            .Executes(context => { /* ... */ return 1; })));
```

인수 포함:

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //illustrative
                return 1;
            })));
```

핵심 사항:

- `args.Dispatcher.Register(...)`을 직접 호출하는 경우에도 명령어가 설치되지만 원장에 들어가지 않으며 `/ncmapi`에도 표시되지 않습니다. 나열하려면 `args.Register`를 사용하세요.

- 명령에는 기본적으로 권한 제한이 없습니다. 필요한 경우 `.Requires(...)`을 직접 추가하세요.

- 실행 시 동작은 전적으로 귀하에게 달려 있습니다. ModApi는 이를 가로채지 않습니다.

### 3.2 등록된 명령어 보기

권한 수준 2가 필요한 내장 `/ncmapi`이 있습니다.

```
/ncmapi
Commands registered via NetCraft-ModApi: 2 total
/ncmapi  list commands registered via NetCraft-ModApi
  /ncmapi
/tpall  teleport all players to the executor
  /tpall
/heal  heal the target players
  /heal <targets>
```

사용법 줄은 명령 트리의 노드 구조에서 즉석에서 계산됩니다. 리터럴은 이름으로 작성되고 인수는 꺾쇠 괄호로 묶이며 자체적으로 실행 가능한 중간 노드도 자체 줄을 얻습니다.

***

## 4. 서버 파사드

이 장의 정면은 모두 `NetCraft.ModApi.Wrapper` 아래에 있습니다. `using NetCraft.ModApi.Wrapper;` 이후에는 사용할 수 있습니다.

Facade는 커널 전체에 분산된 기능을 몇 개의 진입점으로 모으는 `Nc*` 정적 클래스입니다. 커널 인스턴스는 메인 루프가 시작될 때 프로브에 의해 캡처됩니다. `NcServer.IsAvailable`이 false인 경우 아래의 모든 항목이 발생합니다. `Init`가 아닌 이벤트 콜백 내부에서만 사용하세요.

| 외관 | 목적 |
| --- | --- |
| `NcServer` | 서버 인스턴스, 틱 속도, 명령, 엔터티 추적, 플레이어 데이터, 게임 규칙, 브로드캐스트, 명령 실행 |
| `NcPlayers` | 온라인 플레이어 쿼리 및 작업(킥, 텔레포트, 체력, 게임 모드, 권한) |
| `NcWorld` | 오버월드 블록 읽기/쓰기 및 깨짐, 날씨, 시간, 경계선, 시계, 소리, 레벨 이벤트; 다른 차원에 도달하려면 `NcLevel` 핸들을 사용하고 좌표는 일반 `x y z` ints |
| `NcRegistries` | 이름별로 조회되는 내장 레지스트리(블록, 아이템, 유체, 효과, 생물군계, 입자, 개체, 블록 개체) |
| `NcRecipes` | 레시피 쿼리(그리드 제작, 석재 절단, 요리, ID로 레시피 가져오기) |
| `NcLists` | 목록 및 구성(허용 목록, 작전, 금지, `server.properties`) |
| `NcStartup` | 시작 인수(커널 인식되지 않는 토큰 및 이름 기반 구독) |

`NcPlayer`는 정적 파사드가 아니라 **객체 핸들**입니다: `NcPlayers.All` / `Find`가 이를 반환하고, 플레이어 이벤트의 `Player` / `Attacker`도 마찬가지입니다. 핸들은 읽기 전용이며 프로브에 의해 구성됩니다. 모드는 "공개 표면에 커널 유형 없음"의 첫 번째 앵커인 커널의 `ServerPlayer`를 얻을 수 없습니다. 동일한 커널 플레이어는 항상 동일한 핸들에 매핑되며 약한 참조로 내부적으로 캐시되고 플레이어가 로그오프하면 자동으로 무효화됩니다.

`NcLevel`은 레벨에 대해 동일한 모양을 따릅니다. `NcWorld.Overworld` / `Nether` / `End` 및 `NcWorld.Get("minecraft:the_nether")`가 반환되며 `LevelTickArgs.Level`도 하나입니다. 차원 ID, 시간, 날씨, 빌드 높이, 틱 수 및 청크 강제 로딩을 전달합니다. 블록 작업은 `NcWorld`에 유지되고 핸들과 `x y z`을 가져옵니다. `BlockPos`은 결코 표시되지 않으므로 모드의 dll은 커널 수준 유형에 대한 참조를 전달하지 않습니다.

### 4.1 레지스트리

`NcRegistries`은 전체 테이블과 이름별 조회를 모두 제공합니다. 전체 테이블은 반복 및 태그 기반 조회를 위한 것입니다. 이름으로 조회하는 것은 단일 요소를 얻기 위한 것입니다.

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //block default state
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//iterate the whole table
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

레지스트리는 시작 중에 점진적으로 어셈블되고, 어셈블이 완료되기 전에 모드가 로드되므로 `Init`에서 조회된 항목을 캐시하지 마십시오. 어셈블리는 계속 진행 중이며 캐시된 값은 null 참조 또는 오래된 값이 됩니다. 현재 `BuiltInRegistries.BootStrap`은 여전히 ​​빈 구현입니다. 각 레지스트리는 자체 부트스트랩에 의해 별도로 채워지며, 데이터 기반 레지스트리(생물 군계, 레시피 등)에는 데이터 팩 로딩이 연결되기 전에 항목이 거의 없습니다.

### 4.2 레시피

`NcRecipes`은 데이터 팩에서 로드된 레시피 테이블에 의해 지원됩니다. `/reload`은 전체 테이블을 대체하므로 다시 로드할 때 `RecipeHolder`를 유지하지 마세요.

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //compute the output for a crafting grid
    var recipes = NcRecipes.StonecuttingFor(stack);         //stonecutting recipes available for this input
    var smelting = NcRecipes.CookingFor("smelting", stack); //look up by cooking type
    var byId = NcRecipes.Find("minecraft:oak_planks");      //fetch a recipe by id
}
```

***

## 5. 내부

모드를 작성하는 데 이 섹션이 필요하지는 않지만 디버깅할 때 도움이 될 수 있습니다.

### 5.1 프로브

| 수업 | 양식 | 책임 |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)` | 마크 ×4 | 모든 "무슨 일이 일어났습니다" 신호는 하나의 메소드로 유입되고 `label`에 의해 일치하는 이벤트로 전달됩니다 |
| `Internal.CommandProbe.OnCommandsReady(object)` | 콜사이트 | `EffectCommand::Register`에 대한 호출을 대체합니다. 원래 호출을 복원한 후 `CommandRegister` |
| `Internal.PlayerProbe.OnXxx(object, ...)` | 콜사이트 ×6 | 플레이어 이벤트, 후크 포인트당 하나의 방법; 원래 호출을 복원한 후 이벤트를 게시합니다 |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | 콜사이트 ×3 | `ServerChunkCache`의 세 가지 콜백 속성 할당 사이트에 연결된 청크 이벤트 커널에 제어권을 다시 넘기기 전에 래퍼 대리자가 계층화됩니다 |
| `Internal.BlockProbe.OnXxx(...)` | 콜사이트 ×3 | 블록 이벤트; 중단 및 후크 삭제 ​​`ServerBlockUpdates`, 상태 변경이 `IBlockUpdateSink`의 인터페이스 메소드 후크 |

`SignalProbe`의 서명은 `string`만 취하고 `CommandProbe`, `PlayerProbe` 및 `LevelProbe`의 매개 변수는 `object`로 선언됩니다. 이는 의도적인 것입니다. 조립하는 동안 `Lead.Hook`는 대체 메서드의 서명을 확인하고 일단 커널 유형이 나타나면 이를 해결하면 커널 어셈블리를 조기에 가져오고 주입이 해당 창을 놓칩니다. 커널 유형은 코드가 이미 실행 중인 시점인 메서드 본문 내부에만 나타납니다.

서명에서 `object`일 수 없는 유일한 것은 값 유형 매개 변수와 반환 값입니다. `object`은 스택에 대한 참조이고 `float`/`bool`는 값이며 불일치는 잘못된 IL입니다. 따라서 `PlayerProbe.OnHurtPlayer`는 피해량에 대해 `float`를 유지하고 `OnRemovePlayer` 및 `OnHurtPlayer`은 `bool` 반환 값을 유지합니다.

`BlockProbe`은 이 제약 조건의 확장입니다. 블록 위치와 상태는 두 가지 값 유형 `BlockPos`/`BlockState`이며 서명에 그 자체로만 기록될 수 있습니다. 이 두 가지 유형은 `NetCraft.Primitives` 및 `NetCraft.Registry`에서 나오며 둘 다 주입 목록에 없으므로 어셈블리 중에 이를 해결해도 다시 작성할 어셈블리가 조기에 표시되지 않습니다.

### 5.2 후크 포인트 목록

ModApi의 `ncmod.json`에는 2.1의 테이블과 일대일로 일치하는 24개의 규칙이 포함되어 있습니다. 후크 포인트를 변경하거나 규칙을 추가하려면 이 파일을 편집하세요. 편집 후 다시 빌드하고(포함된 리소스임) 결과 dll을 `mods/`에 다시 넣습니다. 후자는 이미 `NetCraft.ModApi.csproj`의 `DeployModToHosts`에 의해 자동으로 수행되었으며 누락되면 규칙이 전혀 적용되지 않는 것으로 나타납니다.

`CommandManager::Execute`에는 하나의 규칙을 공유하는 두 개의 오버로드가 있습니다. `Lead.Hook`의 CallSite는 매개변수 목록이 아닌 "유형 + 메소드 이름"을 기준으로 호출 사이트를 일치시키며 두 오버로드 모두 두 개의 매개변수를 사용하므로 프로브는 첫 번째 매개변수에 대해 `object`를 사용하고 실제 유형으로 디스패치할 수 있습니다.

두 개의 `PacketProcessor` 규칙은 상호보완적입니다. 재생 단계 패킷은 `ScheduleIfPossible`을 거쳐 메인 스레드 큐로 이동하는 반면, 핸드셰이크 및 상태 단계는 즉각적인 처리를 위해 `HandleNow`를 통과합니다. 주어진 패킷은 그 중 하나만 통과합니다. 전자만 후킹하면 핸드셰이크 및 상태 단계가 누락됩니다. 이는 스크립트로 검색하기 가장 쉬운 단계이므로 디버깅하는 동안 이는 "규칙이 적용되지 않았습니다"로 쉽게 오해될 수 있습니다.

세 가지 청크 규칙은 읽기 사이트가 아닌 `ServerChunkCache`의 세 가지 콜백 속성의 **할당 사이트**를 연결합니다. 그 이유는 이 세 가지 속성이 유니캐스트이고 `PersistentServerLevel`이 구성될 때 커널 자체에 의해 이미 점유되어 있기 때문입니다(저장 및 블록 엔터티 정리 논리를 주입함). 모드를 직접 할당하면 커널 복사본이 무시됩니다. 언로드가 지속되지 않고 블록 엔터티가 정리되지 않으며 오류가 전혀 없습니다. 할당 사이트를 후킹하면 그 순간 커널의 콜백과 프로브가 연결될 수 있습니다. 할당은 한 번만 발생하며 이후의 각 트리거는 위임 전달 계층을 하나씩 추가합니다.

`PlayerList::RespawnPlayer`은 비공개이므로 프로브가 원래 호출을 복원할 수 없습니다. 그 사람은 반성을 겪는다(죽을 때마다 한 번 호출되므로 오버헤드는 무시할 수 있음). 또한 커널 정렬을 위한 문이 열려 있습니다. 나중에 `InternalsVisibleTo`이 추가되면 직접 호출로 교체될 수 있습니다.

### 5.3 원장

`Internal.NcCommandRegistry`에는 `args.Register`을 통해 등록된 명령어가 기록됩니다. 이는 원장일 뿐이며 명령 실행에 참여하지 않습니다. 명령 자체는 커널 디스패처에 설치되므로 원장에 문제가 있어도 명령은 계속 작동합니다.

***

## 6. 추가 예정

다음은 위치가 확인되었지만 아직 이벤트가 되지 않은 후크 포인트입니다(목록은 저장소 루트의 `__scan_mod_api.py`에 의해 생성되었습니다).

| 방향 | 후보자 후크 포인트 |
| ----------------- | ---------------------------------------------------------------------------- |
| 엔터티 | `ClientLevel::AddEntity`, `Entity::Die` |
| 세계 | 레벨 로드 및 언로드, `ServerChunkCache` 청크 일괄 처리 |
| 지형 생성 | `ChunkStatus`, `WorldGenRegion::SetBlockState`별 `ChunkGenerator::Generate` 단계 |
| 명령 실행 | `CommandSourceStack::SendSuccess` / `SendFailure` (응답이 절반이고 호출 사이트가 많음) |
| 네트워크 | 패킷 유형별 `ServerGamePacketListenerImpl::HandleXxx` (현재는 통합 진입점만 해당) |

이미 완료된 지침: 레벨 틱은 `ServerEvents.LevelTick`, 저장된 데이터 지속성은 `ServerEvents.SavedDataSaving`, 명령 실행은 `ServerEvents.CommandExecuted`, 블록은 `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`가 되었습니다.

`SetBlock`에 블록 셀을 연결하면 작동하지 않습니다. 여기에는 두 개의 기본 매개변수인 `notifyNeighbors`과 `strict`가 있으므로 컴파일된 호출 사이트는 4~6개의 매개변수를 사용하고 CallSite는 매개변수 목록을 확인하지 않고 "유형 + 메소드 이름"으로 일치하므로 하나의 교체 방법으로 세 가지 스택 모양을 모두 처리할 수 없습니다. 대신 상태 동기화 후크 `IBlockUpdateSink::BlockChanged`(모든 변경 사항을 클라이언트 동기화로 처리하는 `ServerLevel.SetBlock`의 유일한 인터페이스 호출)와 후크 `ServerBlockUpdates`' 자체 메서드를 중단하고 삭제합니다.

엔터티 셀은 인젝션 목록에 없는 `NetCraft.Registry`에 `Entity`이 정의되어 있어 번거롭기 때문에 이를 대상으로 하는 호출 사이트를 다시 작성할 수 없습니다. `ClientLevel::AddEntity`는 클라이언트 어셈블리에 있으며 실행 가능합니다. 사망 이벤트에서는 먼저 `Registry`를 다시 쓸 수 있는지 여부를 확인해야 합니다.

네트워크 셀의 경우 `HandleChat`은 오랫동안 존재해 왔으며 `NetworkEvents.PacketReceived`은 통합된 진입점을 제공하므로 `HandleXxx`별 후크는 훨씬 덜 가치가 있습니다. 패킷 유형별로 세분화된 필터링이 필요한 시나리오만 추가할 가치가 있습니다.

레벨 로드 및 언로드에는 NC 측에 수렴 지점이 없습니다. `DedicatedServer::CreateLevel`은 비공개이므로 원래 호출은 `PlayerList::RespawnPlayer`과 같은 반사를 통해서만 복원될 수 있습니다. 언로드 경로는 훨씬 더 분산되어 있습니다. 이를 수행하려면 먼저 이벤트 인수가 무엇인지 결정해야 합니다.

나머지 것들은 모두 객체 참조(엔티티 인스턴스 등)가 필요하므로 `Mark` 대신 `CallSite`을 사용해야 합니다. 커널 값 유형이 매개변수 사이에 나타나면 대체 메소드의 시그니처에만 그 자체로 기록될 수 있습니다.

인터페이스 표면 목록(`__scan_mod_api.py --api`에서 생성된 인터페이스 표면 목록(`__modapi_api.txt`)도 검토되었습니다. 기능 진입점으로 자격을 갖춘 항목은 도메인별로 4장 파사드에 수집되었으며, 노출되지 않은 나머지는 프로토콜 및 패킷 처리(`Network.Protocol.*`), 렌더링 및 모델(`Client.Render.*`), 지형 생성 및 밀도 기능(`LevelGen.*`)의 세 가지 범주로 분류됩니다. 이것은 커널 내부입니다. 이를 직접 사용하면 모드가 구현 세부 사항에 연결되므로 먼저 커널에서 안정적인 인터페이스를 열어야 합니다.

서버 측에는 Facade로 래핑되지 않은 두 가지가 더 있습니다. `ReloadableServerResources` 인스턴스가 `DedicatedServer`에 매달려 있고 ModApi가 `NetCraft.Server`를 참조하지 않기 때문에 이를 래핑하려면 먼저 커널 기본 클래스에서 속성을 열어야 합니다. `ChunkSender` 및 `ServerWorldBorderListener`는 모드 사용 사례가 없는 내부 흐름입니다.
