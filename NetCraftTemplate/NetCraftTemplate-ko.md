# NetCraft 모드 템플릿

NetCraft 모드 개발용 샘플 코드입니다. `NetCraftTemplate.yaml`의 각 항목은 `examples/` 아래의 파일 하나를 가리키며, `ncm template`이 필요할 때 가져옵니다.

## 사용법

```
ncm template view              모든 항목 나열
ncm template view Wrapper.?    id로 필터링, ?와 *는 와일드카드
ncm template example <api id>  예제 파일 하나를 현재 디렉터리로 가져오기
```

## 두 가지 경로

`NetCraft.ModApi`는 두 개의 네임스페이스를 노출합니다. 맞는 것을 선택하세요:

| 네임스페이스 | 얻는 것 |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | 이벤트, `Nc*` 파사드, `Nc*` 핸들. 공개 표면에 커널 타입이 전혀 나타나지 않으므로 커널 이름이 바뀌어도 모드를 다시 빌드할 필요가 없습니다. |
| `NetCraft.ModApi.Extension` | `[Inject]` 및 `[Mixin]` 특성. 규칙이 커널 타입과 메서드를 직접 지정하므로 더 강력하지만 더 깨지기 쉽습니다. |

## 핸들

`NcPlayer`와 `NcLevel`은 읽기 전용 핸들입니다. 이들이 돌려주는 모든 것은 일반 문자열, 숫자 또는 불리언(`NcLevel.Dimension`, `NcPlayer.X`)이며, 커널 타입은 결코 아닙니다. 따라서 커널 이름이 바뀌어도 모드를 다시 빌드할 필요가 없습니다. 블록 작업은 레벨 핸들에 `x y z`를 더해 받습니다.

## 일반적인 호출

모드 프로젝트 안에서 이 패널을 열면, 아래 호출 중 코드가 실제로 사용하는 모든 호출이 `NetCraftTemplate.yaml`에 대해 판정됩니다: 멤버가 선언되어 있으면 초록, 멤버가 없으면 노랑, 타입이 아예 선언되지 않았으면 빨강. 강조된 이름 위에 마우스를 올리면 이유를 볼 수 있습니다.

| 호출 | 하는 일 |
| --- | --- |
| `NcServer.IsAvailable` | 서버가 실행 중이고 캡처되었는지 여부 |
| `NcServer.Broadcast` | 온라인 모두에게 보내는 시스템 메시지 |
| `NcServer.Execute` | 콘솔로서 명령 실행 |
| `NcWorld.GetBlock` | 블록 하나 읽기, 청크가 로드되지 않았으면 null |
| `NcWorld.SetBlock` | 블록 하나 쓰기, 전체 업데이트 체인 실행 |
| `NcWorld.BreakBlock` | 플레이어가 하는 방식으로 블록 파괴 |
| `NcWorld.Overworld` | 오버월드 레벨 핸들 |
| `NcLevel.Dimension` | 레벨 핸들의 차원 id, 예: minecraft:overworld |
| `NcLevel.DayTime` | 한 차원의 시간 읽기 또는 설정 |
| `NcPlayer.Name` | 플레이어 이름 |
| `NcPlayer.Health` | 현재 체력 |
| `NcPlayers.Find` | 이름으로 온라인 플레이어 찾기 |
| `NcPlayers.Send` | 비공개 시스템 메시지 |
| `NcRegistries.FindState` | 네임스페이스 id로 블록 상태 찾기 |
| `NcRegistries.FindItem` | 네임스페이스 id로 아이템 찾기 |
| `ServerEvents.Tick` | 모든 서버 tick마다 실행 |
| `ServerEvents.PlayerJoin` | 플레이어가 참가를 마침 |
| `ServerEvents.BlockBroken` | 블록이 실제로 교체됨 |