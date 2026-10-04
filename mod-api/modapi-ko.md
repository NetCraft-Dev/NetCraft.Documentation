# NetCraft-ModApi 레퍼런스

`NetCraft.ModApi`는 NetCraft가 모드에 노출하는 API 표면입니다. 두 가지 정체성을 가집니다: 여러분에게는 API 라이브러리이고, 그 자체로는 일반 모드입니다(`id`는 `netcraft-modapi`이며, 자체 `ncmod.json`과 주입 프로브를 함께 제공합니다).

이 파일은 API가 성장함에 따라 함께 늘어납니다. 아키텍처 배경, Fabric과의 차이점, 모드 작성 방법은 [modding-guide-ko.md](modding-guide-ko.md)를 참고하세요.

- 어셈블리: `NetCraft.ModApi.dll`

- 종속성: `NetCraft`(메인 라이브러리), `NetCraft.Game`

공개 표면은 세 개의 네임스페이스로 나뉩니다:

| 네임스페이스 | 내용 | 비고 |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | 이벤트 및 구독 기본 클래스 `NcEvent<T>`, `Nc*` 파사드, `Nc*` 객체 핸들 | 래퍼 계층; 공개 표면에 커널 타입이 없음 |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 애노테이션 | 확장 지점; 규칙이 커널 클래스와 메서드 이름에 결합됨 |
| `NetCraft.ModApi.Internal` | 주입 프로브 | 직접 참조하지 마세요 |

루트 네임스페이스 `NetCraft.ModApi`에는 엔트리 클래스 `ModApiEntry`만 들어 있습니다. `Wrapper`와 `Extension`은 나란히 존재하는 두 경로입니다. 선택 방법은 [modding-guide-ko.md 2.9](modding-guide-ko.md#29-두-가지-경로-래퍼-계층과-확장-지점)를 참고하세요.

***

## 1. 빠른 시작

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //구독은 핸들을 반환하며, 핸들을 Dispose하면 등록이 해제됩니다
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //핸들로 다른 작업을 할 수도 있습니다
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

`ncmod.json`에서 `entry`를 이 클래스로 지정하고 `hooks`는 비워 두세요. 아래 이벤트들은 모두 ModApi 자체 프로브가 제공합니다.

***

## 2. 이벤트

모든 이벤트는 `NetCraft.ModApi.Wrapper` 아래에 있습니다. `using NetCraft.ModApi.Wrapper;`를 추가하면 사용할 수 있습니다.

### 2.1 요약 표

| 이벤트                           | Args 타입              | 트리거                                                   | 측   | ModApi 훅 지점                                        |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick`             | `ServerTickArgs`       | 서버 메인 루프의 매 틱                        | 서버 | `DedicatedServer::Tick` (Mark)                           |
| `ServerEvents.Started`          | `ServerPhaseArgs`      | `Done (x.xxxs)!`가 출력된 뒤 메인 루프 시작      | 서버 | `MinecraftServer::Run` (Mark)                            |
| `ServerEvents.Stopping`         | `ServerPhaseArgs`      | 서버가 종료를 시작함; 플레이어가 곧 연결 해제됨 | 서버 | `DedicatedServer::Stop` (Mark)                    |
| `ServerEvents.CommandRegister`  | `CommandRegisterArgs`  | 모든 내장 명령이 등록됨                | 서버 | `EffectCommand::Register` 호출 지점 (CallSite)           |
| `ServerEvents.PlayerJoin`       | `PlayerJoinArgs`       | 접속 패킷 시퀀스가 전송됨                    | 서버 | `PlayerList::PlaceNewPlayer` 호출 지점 (CallSite)        |
| `ServerEvents.PlayerLeave`      | `PlayerLeaveArgs`      | 플레이어가 온라인 목록에서 제거됨                       | 서버 | `PlayerList::RemovePlayer` 호출 지점 (CallSite)          |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | 연결 해제 패킷이 전송되고 연결이 닫힘              | 서버 | `ServerPlayer::Disconnect` 호출 지점 (CallSite)          |
| `ServerEvents.PlayerHurt`       | `PlayerHurtArgs`       | 실제로 피해가 가해짐                                     | 서버 | `PlayerList::HurtPlayer` 호출 지점 (CallSite)            |
| `ServerEvents.PlayerDeath`      | `PlayerDeathArgs`      | 체력이 0으로 재설정된 직후                   | 서버 | `PlayerList::RespawnPlayer` 호출 지점 (CallSite)         |
| `ServerEvents.PlayerChat`       | `PlayerChatArgs`       | 채팅 브로드캐스트 이후                                          | 서버 | `ServerGamePacketListenerImpl::HandleChat` 호출 지점 (CallSite) |
| `ServerEvents.ChunkLoaded`      | `ChunkLoadedArgs`      | 청크가 처음으로 메모리에 들어옴                    | 서버 | `ServerChunkCache::set_ChunkLoaded` 할당 지점 (CallSite) |
| `ServerEvents.ChunkUnloaded`    | `ChunkUnloadedArgs`    | 청크가 메모리에서 나감                                       | 서버 | `ServerChunkCache::set_ChunkUnloaded` 할당 지점 (CallSite) |
| `ServerEvents.ChunkSaved`       | `ChunkSavedArgs`       | 청크가 디스크에 기록되기 전에 스냅샷이 생성됨        | 서버 | `ServerChunkCache::set_ChunkSaveSink` 할당 지점 (CallSite) |
| `ServerEvents.CommandExecuted`  | `CommandExecutedArgs`  | 명령 실행이 끝남; 구문 오류와 권한 거부도 포함됨 | 서버 | `CommandManager::Execute` (CallSite) |
| `ServerEvents.LevelTick`        | `LevelTickArgs`        | 레벨 틱, 틱마다 로드된 레벨별로 한 번                | 서버 | `PersistentServerLevel::Tick` (CallSite)                 |
| `ServerEvents.SavedDataSaving`  | `SavedDataSavingArgs`  | 저장 데이터가 디스크에 기록됨, 청크 저장보다 한 단계 늦음 | 서버 | `SavedDataStorage::ScheduleSave` (CallSite)          |
| `ServerEvents.BlockChanged`     | `BlockChangedArgs`     | 블록 상태가 변경되어 클라이언트로 동기화되기 직전             | 서버 | `IBlockUpdateSink::BlockChanged` (CallSite)              |
| `ServerEvents.BlockBroken`      | `BlockBrokenArgs`      | 블록이 부서짐; 플레이어 채굴과 레드스톤 자폭 모두 포함 | 서버 | `ServerBlockUpdates::BreakBlock` (CallSite) |
| `ServerEvents.ItemDropped`      | `ItemDroppedArgs`      | 드롭된 아이템 엔티티가 생성됨, 블록 파괴 드롭과 조리 결과물 포함 | 서버 | `ServerBlockUpdates::SpawnDrop` (CallSite) |
| `NetworkEvents.PacketReceived`  | `PacketReceivedArgs`   | 핸들러에 큐잉된 모든 인바운드 패킷, 핸드셰이크와 상태 단계 포함 | 양쪽   | `PacketProcessor::ScheduleIfPossible` 및 `HandleNow` (CallSite) |
| `ClientEvents.Tick`             | `ClientTickArgs`       | 클라이언트 메인 루프의 매 틱                        | 클라이언트 | `MinecraftClient::Tick` (Mark)                           |

플레이어 이벤트의 순서: 죽음은 피해 흐름에 중첩되므로 `PlayerDeath`가 해당 `PlayerHurt`보다 먼저 발생합니다. `PlayerLeave`와 `PlayerDisconnect`는 서로 다른 두 가지입니다 — 전자는 온라인 목록에서의 제거를 의미하며(`/kick` 이후에도 연결이 끊겨야만 발생합니다), 후자는 연결이 끊기는 것 자체를 의미합니다. 두 이벤트가 항상 짝을 이루어 나타난다고 보장할 수 없습니다.

### 2.2 구독과 등록 해제

```csharp
IDisposable Subscribe(Action<T> handler)
```

- 같은 이벤트를 여러 번 구독하면 구독 순서대로 알림이 전달됩니다.

- 디스패치는 콜백 목록의 스냅샷을 사용하므로, 콜백 내부에서 구독하거나 등록 해제해도 현재 디스패치에는 영향을 주지 않습니다.

- 등록을 해제하지 않으면 영구히 유효합니다. 모드는 언로드 메커니즘을 제공하지 않으므로 수동 등록 해제는 대개 필요하지 않습니다.

### 2.3 Args 타입

**`ServerTickArgs`**

| 속성    | 타입   | 비고                                |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | 이번 실행 이후의 틱 수, 1부터 시작 |

이는 커널의 `TickCount`가 아니라 ModApi 자체가 집계한 값입니다.

**`ClientTickArgs`**

| 속성    | 타입   | 비고                           |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | 위와 같으며, 클라이언트 측에서 독립적으로 집계됨 |

**`ServerPhaseArgs`**

| 속성 | 타입     | 비고                        |
| -------- | -------- | ---------------------------- |
| `Phase`  | `string` | 단계 이름, `started` 또는 `stopping` |

이 필드는 이벤트 자체와 중복되지만, 로깅이 하나의 통일된 형식을 쓰도록 유지됩니다.

**`CommandRegisterArgs`**

| 멤버                               | 타입                                    | 비고            |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher`                         | `CommandDispatcher<CommandSourceStack>` | 커널의 명령 디스패처 |
| `Register(name, description, build)` | 메서드                                  | 명령을 등록하고 원장에 기록합니다, 3.1 참고 |

**`PlayerJoinArgs`**

| 속성      | 타입           | 비고                                          |
| ------------- | -------------- | ---------------------------------------------- |
| `Player`      | `ServerPlayer` | 방금 접속한 플레이어; 접속 패킷이 이미 전송되어 상태를 안전하게 읽을 수 있음 |
| `ProfileName` | `string`       | 플레이어 이름                                    |

**`PlayerLeaveArgs`**

| 속성  | 타입           | 비고                                 |
| --------- | -------------- | ------------------------------------- |
| `Player`  | `ServerPlayer` | 나가는 플레이어, 이 시점에는 더 이상 온라인 목록에 없음 |
| `Removed` | `bool`         | 실제로 제거되었는지 여부; 반복 제거 시 `false` |

**`PlayerDisconnectArgs`**

| 속성 | 타입           | 비고                                 |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | 연결이 끊긴 플레이어               |
| `Reason` | `string`       | 연결 해제 사유; 컴포넌트로 주어지면 일반 텍스트 |

연결 해제 패킷이 이미 전송되고 연결이 닫혔으므로, 이 플레이어에게 패킷을 보내도 이제 효과가 없습니다.

**`PlayerHurtArgs`**

| 속성   | 타입            | 비고                                        |
| ---------- | --------------- | -------------------------------------------- |
| `Player`   | `ServerPlayer`  | 피해를 입은 플레이어                      |
| `Attacker` | `ServerPlayer?` | 피해를 가한 플레이어; 환경 피해와 명령 피해의 경우 `null` |
| `Amount`   | `float`         | 이번 피해량                      |

무적 프레임 중이나 사망 후에는 발생하지 않습니다(커널의 `Hurt`가 `false`를 반환).

**`PlayerDeathArgs`**

| 속성   | 타입            | 비고                          |
| ---------- | --------------- | ------------------------------ |
| `Player`   | `ServerPlayer`  | 사망한 플레이어            |
| `Attacker` | `ServerPlayer?` | 살해자; 없으면 `null` |

커널은 체력이 0이 된 직후 재설정하므로, 이벤트가 발생할 때 플레이어는 이미 리스폰 지점에서 최대 체력 상태입니다. 사망 순간의 좌표와 드롭은 얻을 수 없습니다.

**`PlayerChatArgs`**

| 속성     | 타입     | 비고                |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | 발신자 이름          |
| `Message`    | `string` | 일반 텍스트 메시지   |

이 이벤트는 **읽기 전용 알림**입니다: 원본 메서드가 이미 메시지를 브로드캐스트했으므로 여기서 `Message`를 바꿔도 효과가 없습니다.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

세 이벤트는 동일한 args 형태를 공유합니다:

| 속성 | 타입  | 비고              |
| -------- | ----- | ------------------ |
| `X`      | `int` | 청크 좌표 X |
| `Z`      | `int` | 청크 좌표 Z |

가장 실수하기 쉬운 것은 `ChunkSaved`입니다: 커널은 이 콜백이 **동기적으로 스냅샷을 생성**할 것을 요구하는 반면, 직렬화와 디스크 쓰기는 커널 자체가 비동기로 수행합니다. 따라서 이 콜백에서 시간이 많이 걸리는 작업을 하면 청크 언로드가 직접 느려지고, 파괴적 작업(블록 제거, 인벤토리 변경)도 여기에 두어서는 안 됩니다 — 이 콜백은 스냅샷 시점만 보장합니다.

`ChunkUnloaded`가 발생할 때 블록 엔티티는 이미 청크와 함께 정리된 상태입니다. 블록을 읽고 싶다면 `ChunkSaved`(이 역시 블록에 접근할 수는 없습니다)나 더 이른 시점을 사용하세요.

**`CommandExecutedArgs`**

| 속성  | 타입                   | 비고                                 |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string`               | 원시 명령 텍스트; 채팅 명령에는 앞의 슬래시가 없음 |
| `Result`  | `int`                  | 명령 반환값; 0은 실패 또는 거부를 의미 |
| `Source`  | `CommandSourceStack?`  | 명령 소스; 플레이어 오버로드 경로에서는 `null` |
| `Player`  | `ServerPlayer?`        | 명령을 실행한 플레이어; 콘솔에서 실행한 경우 `null` |

이 이벤트는 명령이 끝난 **후에** 발생하며 실행을 변경할 수 없습니다. 구문 오류와 권한 거부도 여기로 들어오므로 `Result`로 구분하세요. 플레이어가 채팅창에서 보낸 명령은 `Execute(ServerPlayer, string)` 오버로드를 거치며, 이때 커널이 내부적으로 명령 소스를 만들기 때문에 `Source`는 `null`이고 `Player`만 설정됩니다.

**`LevelTickArgs`**

| 속성       | 타입                    | 비고                                        |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level`        | `NcLevel`               | 이번 틱에 진행된 레벨                 |
| `RunsNormally` | `bool`                  | 정상적으로 진행되었는지 여부; `/tick freeze` 중에는 `false` |

틱마다 로드된 레벨별로 한 번 발생하므로, 다중 레벨 월드는 틱마다 여러 번 받습니다. 트리거 지점은 레벨 틱이 **완료된** 후이며, 가로채기 지점이 아니라 관찰 지점입니다.

**`SavedDataSavingArgs`**

| 속성  | 타입                | 비고                       |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage`  | 저장 중인 저장 데이터 테이블 |

이것은 `ServerEvents.ChunkSaved`와 다른 경로입니다: 청크 쪽은 스냅샷만 생성하고 비동기로 쓰는 반면, 이쪽은 동기 쓰기가 완료된 후에 발생합니다. 월드 시계, 게임 규칙, 월드 경계 데이터가 여기를 거칩니다.

**`BlockChangedArgs`**

| 속성 | 타입         | 비고                       |
| -------- | ------------ | --------------------------- |
| `Pos`    | `BlockPos`   | 변경된 블록의 위치 |
| `State`  | `BlockState` | 변경 후의 블록 상태  |

상태는 이미 청크에 기록되었고 클라이언트로 동기화되기 직전이므로, 여기서 변경 자체를 수정할 수 없습니다. 레드스톤 컴포넌트가 동작 콜백에서 자신의 상태를 바꾸는 것도 이 경로로 나갑니다. 빈도가 높으므로 콜백에서 시간이 많이 걸리는 작업을 하지 마세요.

**`BlockBrokenArgs`**

| 속성 | 타입            | 비고                                      |
| -------- | --------------- | ------------------------------------------ |
| `Pos`    | `BlockPos`      | 부서진 블록의 위치                |
| `Player` | `ServerPlayer?` | 부순 주체; 레드스톤 등 플레이어가 아닌 원인의 경우 `null` |

블록이 실제로 교체될 때만 발생합니다. 빈 위치나 거부된 파괴는 트리거하지 않습니다. 파괴 효과와 드롭은 이미 처리되었으므로 이벤트에서 읽는 것은 결과입니다.

**`ItemDroppedArgs`**

| 속성 | 타입        | 비고                   |
| -------- | ----------- | ----------------------- |
| `Pos`    | `BlockPos`  | 드롭된 아이템이 나타난 위치 |
| `Stack`  | `ItemStack` | 드롭된 아이템 스택   |

블록 파괴 드롭과 모닥불 조리 결과물이 모두 여기를 거칩니다. 빈 아이템 스택은 엔티티를 생성하지 않으므로 이벤트가 없습니다.

**`PacketReceivedArgs`**

| 속성        | 타입     | 비고                    |
| --------------- | -------- | ------------------------ |
| `Listener`      | `object` | 이 패킷을 받는 리스너 |
| `Packet`        | `object` | 패킷 객체 자체  |
| `IsServerbound` | `bool`   | 서버바운드 패킷인지 여부 |

모든 인바운드 패킷에 대해 발생하며, 핸드셰이크, 상태, 구성, 플레이의 네 단계를 모두 포괄합니다. 이동 패킷은 틱마다 여러 번 도착하므로 콜백에서 시간이 많이 걸리는 작업을 하지 마세요. 패킷은 이미 객체로 디코딩되었지만 비즈니스 계층에 들어가기 전이므로, 타입을 구분하려면 `Packet`을 직접 확인하세요. 아웃바운드 패킷은 이 이벤트의 범위 밖입니다.

***

## 3. 확장 지점

### 3.1 명령 등록

시점은 `ServerEvents.CommandRegister`입니다. 이 이벤트의 args를 캐시하지 마세요. 내부 명령 트리는 시작 시 한 번만 만들어집니다.

```csharp
public void Register(
    string name,                                          //명령 리터럴, 슬래시 없이
    string description,                                   //한 줄 설명, /ncmapi 원장에 표시됨
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //인수와 실행기를 붙임
```

받은 `build`는 커널의 brigadier 빌더입니다. 인수, 하위 명령, 권한 술어를 커널 방식대로 작성하세요:

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //권한 술어
            .Executes(context => { /* ... */ return 1; })));
```

인수를 사용하는 경우:

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //예시
                return 1;
            })));
```

핵심 사항:

- `args.Dispatcher.Register(...)`를 직접 호출해도 명령은 설치되지만 원장에 들어가지 않으므로 `/ncmapi`에 표시되지 않습니다. 목록에 넣고 싶으면 `args.Register`를 사용하세요.

- 명령은 기본적으로 권한 제한이 없으므로 필요하면 `.Requires(...)`를 직접 추가하세요.

- 실행 시의 동작은 전적으로 여러분에게 달려 있으며 ModApi는 개입하지 않습니다.

### 3.2 등록된 명령 보기

권한 레벨 2가 필요한 내장 `/ncmapi`가 있습니다:

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

사용법 줄은 명령 트리의 노드 구조로부터 즉석에서 계산됩니다: 리터럴은 이름으로 표기되고, 인수는 꺾쇠괄호로 감싸지며, 그 자체로 실행 가능한 중간 노드도 별도의 줄을 가집니다.

***

## 4. 서버 파사드

이 장의 파사드는 모두 `NetCraft.ModApi.Wrapper` 아래에 있습니다. `using NetCraft.ModApi.Wrapper;`를 추가하면 사용할 수 있습니다.

파사드는 커널 전반에 흩어진 기능을 몇 개의 진입점으로 모은 `Nc*` 정적 클래스입니다. 커널 인스턴스는 메인 루프가 시작될 때 프로브가 캡처합니다. `NcServer.IsAvailable`이 false이면 아래의 모든 것이 예외를 던집니다 — `Init`이 아니라 이벤트 콜백 안에서만 사용하세요.

| 파사드 | 용도 |
| --- | --- |
| `NcServer` | 서버 인스턴스, 틱 레이트, 명령, 엔티티 추적, 플레이어 데이터, 게임 규칙, 브로드캐스트, 명령 실행 |
| `NcPlayers` | 접속 중인 플레이어 조회와 조작(킥, 텔레포트, 체력, 게임 모드, 권한) |
| `NcWorld` | 오버월드 블록 읽기/쓰기와 파괴, 날씨, 시간, 경계, 시계, 사운드, 레벨 이벤트; 다른 차원에 접근하려면 `NcLevel` 핸들을 사용하며, 좌표는 평범한 `x y z` 정수 |
| `NcRegistries` | 이름으로 조회하는 내장 레지스트리(블록, 아이템, 유체, 효과, 생물군계, 입자, 엔티티, 블록 엔티티) |
| `NcRecipes` | 레시피 조회(그리드 제작, 석재 절단, 조리; id로 레시피 가져오기) |
| `NcLists` | 목록과 설정(화이트리스트, ops, 밴, `server.properties`) |
| `NcStartup` | 시작 인수(커널이 인식하지 못한 토큰과 이름 기반 구독) |

`NcPlayer`는 정적 파사드가 아니라 **객체 핸들**입니다: `NcPlayers.All` / `Find`가 이를 반환하며, 플레이어 이벤트의 `Player` / `Attacker`도 바로 이것입니다. 핸들은 읽기 전용이며 프로브가 생성합니다. 모드는 커널의 `ServerPlayer`를 얻을 수 없습니다 — 이것이 "공개 표면에 커널 타입이 없음"의 첫 번째 근거입니다. 동일한 커널 플레이어는 항상 동일한 핸들로 매핑되며, 내부적으로 약한 참조로 캐시되고 플레이어가 로그오프하면 자동으로 무효화됩니다.

`NcLevel`은 레벨에 대해 같은 형태를 따릅니다. `NcWorld.Overworld` / `Nether` / `End`와 `NcWorld.Get("minecraft:the_nether")`가 이를 반환하며, `LevelTickArgs.Level`도 그중 하나입니다. 차원 id, 시간, 날씨, 빌드 높이, 틱 수, 청크 강제 로딩을 담고 있습니다. 블록 조작은 `NcWorld`에 남아 있으며 핸들과 `x y z`를 받습니다. `BlockPos`는 결코 나타나지 않으므로 모드의 dll은 커널 레벨 타입에 대한 참조를 담지 않습니다.

### 4.1 레지스트리

`NcRegistries`는 전체 테이블과 이름별 조회를 모두 제공합니다. 전체 테이블은 반복과 태그 기반 조회용이고, 이름별 조회는 단일 요소를 얻기 위한 것입니다:

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //블록 기본 상태
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//전체 테이블 순회
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

레지스트리는 시작 중에 점진적으로 구성되며, 모드는 구성이 완료되기 전에 로드됩니다. 따라서 `Init`에서 조회한 것을 캐시하지 마세요 — 구성이 아직 진행 중이라 캐시된 값은 null 참조이거나 오래된 값이 됩니다. 현재 `BuiltInRegistries.BootStrap`은 여전히 빈 구현입니다. 각 레지스트리는 자체 Bootstrap이 개별적으로 채우며, 데이터 기반 레지스트리(생물군계, 레시피 등)는 데이터 팩 로딩이 연결되기 전까지 항목이 매우 적습니다.

### 4.2 레시피

`NcRecipes`는 데이터 팩에서 로드한 레시피 테이블로 뒷받침됩니다. `/reload`는 전체 테이블을 교체하므로 리로드를 넘겨 `RecipeHolder`를 보관하지 마세요.

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //제작 그리드의 출력 계산
    var recipes = NcRecipes.StonecuttingFor(stack);         //이 입력에 사용할 수 있는 석재 절단 레시피
    var smelting = NcRecipes.CookingFor("smelting", stack); //조리 유형으로 조회
    var byId = NcRecipes.Find("minecraft:oak_planks");      //id로 레시피 가져오기
}
```

***

## 5. 내부 구조

모드 작성에는 이 장이 필요하지 않지만, 디버깅할 때 도움이 될 수 있습니다.

### 5.1 프로브

| 클래스                                                    | 형태        | 역할                                              |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)`                  | Mark ×4     | 모든 "무언가 일어남" 신호가 하나의 메서드로 모여 `label`에 따라 해당 이벤트로 디스패치됨 |
| `Internal.CommandProbe.OnCommandsReady(object)`          | CallSite    | `EffectCommand::Register` 호출을 교체함; 원본 호출을 복원한 뒤 `CommandRegister`를 발생시킴 |
| `Internal.PlayerProbe.OnXxx(object, ...)`                | CallSite ×6 | 플레이어 이벤트, 훅 지점당 메서드 하나; 원본 호출을 복원한 뒤 이벤트를 발행함 |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | CallSite ×3 | 청크 이벤트, `ServerChunkCache`의 세 콜백 속성 할당 지점에 훅됨; 커널에 제어를 돌려주기 전에 래퍼 델리게이트를 덧씌움 |
| `Internal.BlockProbe.OnXxx(...)`                         | CallSite ×3 | 블록 이벤트; 파괴와 드롭은 `ServerBlockUpdates`에 훅하고, 상태 변경은 `IBlockUpdateSink`의 인터페이스 메서드에 훅함 |

`SignalProbe`의 시그니처는 `string`만 받으며, `CommandProbe`, `PlayerProbe`, `LevelProbe`의 매개변수는 `object`로 선언됩니다 — 이는 의도적입니다: 어셈블리 구성 중에 `Lead.Hook`이 대체 메서드의 시그니처를 해석하는데, 여기에 커널 타입이 나타나면 그것을 해석하면서 커널 어셈블리를 조기에 끌어올려 주입이 시점을 놓치게 됩니다. 커널 타입은 메서드 본문 안에서만 나타나며, 그때는 코드가 이미 실행 중입니다.

시그니처에서 `object`가 될 수 없는 유일한 것은 값 타입 매개변수와 반환값입니다: `object`는 스택에서 참조이고 `float`/`bool`은 값이므로, 불일치는 유효하지 않은 IL입니다. 그래서 `PlayerProbe.OnHurtPlayer`는 피해량에 `float`을 유지하고, `OnRemovePlayer`와 `OnHurtPlayer`는 `bool` 반환값을 유지합니다.

`BlockProbe`는 이 제약의 연장입니다: 블록 위치와 상태는 `BlockPos`/`BlockState`라는 두 값 타입이며, 시그니처에 자기 자신으로만 쓸 수 있습니다. 이 두 타입은 `NetCraft.Primitives`와 `NetCraft.Registry`에서 오는데, 둘 다 주입 목록에 없으므로 어셈블리 구성 중에 해석해도 재작성 대상 어셈블리를 조기에 끌어올리지 않습니다.

### 5.2 훅 지점 목록

ModApi의 `ncmod.json`에는 스물네 개의 규칙이 있으며, 2.1의 표와 일대일로 대응합니다. 훅 지점을 바꾸거나 규칙을 추가하려면 이 파일을 편집하세요. 편집 후에는 다시 빌드하고(임베디드 리소스입니다) 결과 dll을 `mods/`에 다시 넣으세요 — 후자는 `NetCraft.ModApi.csproj`의 `DeployModToHosts`가 이미 자동으로 처리하며, 이를 빠뜨리면 규칙이 전혀 적용되지 않는 것으로 나타납니다.

`CommandManager::Execute`에는 하나의 규칙을 공유하는 두 개의 오버로드가 있습니다. `Lead.Hook`의 CallSite는 매개변수 목록이 아니라 "타입 + 메서드 이름"으로 호출 지점을 매칭하며, 두 오버로드 모두 매개변수 두 개를 받으므로 프로브가 첫 번째 매개변수에 `object`를 받고 실제 타입으로 디스패치할 수 있습니다.

두 개의 `PacketProcessor` 규칙은 상호 보완적입니다: 플레이 단계 패킷은 `ScheduleIfPossible`을 거쳐 메인 스레드 큐로 들어가고, 핸드셰이크와 상태 단계는 `HandleNow`를 거쳐 즉시 처리됩니다. 어느 패킷이든 둘 중 하나만 통과합니다. 전자만 훅하면 핸드셰이크와 상태 단계를 놓치는데, 하필 이 단계가 스크립트로 가장 프로브하기 쉬워서 디버깅 중에 "규칙이 적용되지 않았다"고 오해하기 쉽습니다.

세 개의 청크 규칙은 `ServerChunkCache`의 세 콜백 속성의 **할당 지점**에 훅하며, 읽기 지점이 아닙니다. 그 이유는 그 세 속성이 유니캐스트이고 `PersistentServerLevel`이 생성될 때 커널 자체가 이미 점유하기 때문입니다(저장 및 블록 엔티티 정리 로직을 주입합니다). 모드가 직접 할당하면 커널의 복사본을 덮어써서 언로드가 저장되지 않고, 블록 엔티티가 정리되지 않으며, 아무 오류도 나지 않습니다. 할당 지점에 훅하면 그 시점에 커널 콜백과 프로브를 연결할 수 있습니다. 할당은 한 번만 일어나고, 이후 트리거마다 델리게이트 전달 계층이 하나씩 추가됩니다.

`PlayerList::RespawnPlayer`는 private이라 프로브가 원본 호출을 복원할 수 없습니다. 이 하나는 리플렉션을 거칩니다(사망마다 한 번 호출되므로 오버헤드는 무시할 만합니다). 이는 커널 정렬을 위한 여지도 남깁니다: 나중에 이에 대해 `InternalsVisibleTo`가 추가되면 직접 호출로 바꿀 수 있습니다.

### 5.3 원장

`Internal.NcCommandRegistry`는 `args.Register`로 등록된 명령을 기록합니다. 이것은 원장일 뿐이며 명령 실행에 관여하지 않습니다. 명령 자체는 커널 디스패처에 설치되므로 원장에 문제가 있어도 명령은 여전히 작동합니다.

***

## 6. 추가 예정

다음은 위치는 확인되었지만 아직 이벤트가 되지 않은 훅 지점입니다(목록은 저장소 루트의 `__scan_mod_api.py`가 생성했습니다):

| 방향         | 후보 훅 지점                                                        |
| ----------------- | ---------------------------------------------------------------------------- |
| 엔티티          | `ClientLevel::AddEntity`, `Entity::Die`                                      |
| 월드             | 레벨 로드와 언로드, `ServerChunkCache` 청크 배칭                     |
| 지형 생성 | `ChunkStatus`별 `ChunkGenerator::Generate` 단계, `WorldGenRegion::SetBlockState` |
| 명령 실행 | `CommandSourceStack::SendSuccess` / `SendFailure`(응답 절반으로, 호출 지점이 많음) |
| 네트워크           | 패킷 타입별 `ServerGamePacketListenerImpl::HandleXxx`(현재는 통합 진입점만 있음) |

이미 완료된 방향: 레벨 틱은 `ServerEvents.LevelTick`이 되었고, 저장 데이터 영속화는 `ServerEvents.SavedDataSaving`이 되었으며, 명령 실행은 `ServerEvents.CommandExecuted`가 되었고, 블록은 `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`가 되었습니다.

블록 칸을 `SetBlock`에 훅하는 것은 통하지 않습니다: `notifyNeighbors`와 `strict`라는 두 기본 매개변수가 있어 컴파일된 호출 지점은 4~6개의 매개변수를 받으며, CallSite는 매개변수 목록을 보지 않고 "타입 + 메서드 이름"으로 매칭하므로 하나의 대체 메서드가 세 가지 스택 형태를 모두 처리할 수 없습니다. 대신 두 곳에 훅합니다: 상태 동기화는 `IBlockUpdateSink::BlockChanged`에 훅하고(`ServerLevel.SetBlock`의 유일한 인터페이스 호출로, 클라이언트 동기화가 있는 모든 변경을 포괄함), 파괴와 드롭은 `ServerBlockUpdates` 자체 메서드에 훅합니다.

엔티티 칸은 `Entity`가 `NetCraft.Registry`에 정의되어 있고 이 어셈블리가 주입 목록에 없어서 까다롭습니다. 그를 대상으로 하는 호출 지점은 재작성할 수 없습니다. `ClientLevel::AddEntity`는 클라이언트 어셈블리에 있어 가능하며, 사망 이벤트는 먼저 `Registry`를 재작성할 수 있는지 해결해야 합니다.

네트워크 칸의 경우 `HandleChat`이 오래전부터 있었고 `NetworkEvents.PacketReceived`가 통합 진입점을 제공하므로, `HandleXxx`별 훅은 가치가 훨씬 떨어집니다. 패킷 타입별로 세밀하게 필터링해야 하는 시나리오만 추가할 가치가 있습니다.

레벨 로드와 언로드는 NC 쪽에 수렴 지점이 없습니다: `DedicatedServer::CreateLevel`이 private이라 `PlayerList::RespawnPlayer`처럼 리플렉션으로만 원본 호출을 복원할 수 있고, 언로드 경로는 훨씬 더 흩어져 있습니다. 하려면 먼저 이벤트 args가 무엇이어야 하는지 정해야 합니다.

나머지는 모두 객체 참조(엔티티 인스턴스 등)가 필요하므로 `Mark`가 아니라 `CallSite`를 사용해야 합니다. 매개변수 중에 커널 값 타입이 나타나면 대체 메서드의 시그니처에 자기 자신으로만 쓸 수 있습니다.

인터페이스 표면 목록(`__modapi_api.txt`, `__scan_mod_api.py --api`가 생성)도 검토했습니다: 기능 진입점 자격을 갖춘 항목은 도메인별로 4장의 파사드로 모았고, 노출되지 않은 나머지는 세 범주로 나뉩니다 — 프로토콜과 패킷 처리(`Network.Protocol.*`), 렌더링과 모델(`Client.Render.*`), 지형 생성과 밀도 함수(`LevelGen.*`)입니다. 이들은 커널 내부이므로 직접 사용하면 모드가 구현 세부 사항에 묶입니다. 따라서 먼저 커널에 안정적인 인터페이스를 열어야 합니다.

서버 쪽에는 파사드로 감싸지지 않은 것이 두 가지 더 있습니다: `ReloadableServerResources` 인스턴스는 `DedicatedServer`에 매달려 있는데, ModApi가 `NetCraft.Server`를 참조하지 않으므로 감싸려면 먼저 커널 기본 클래스에 속성을 열어야 합니다. `ChunkSender`와 `ServerWorldBorderListener`는 모드가 쓸 용도가 없는 내부 흐름입니다.
