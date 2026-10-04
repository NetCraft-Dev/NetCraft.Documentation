# NetCraft 모드 템플릿

NetCraft 모드 개발용 샘플 코드입니다. `NetCraftTemplate.yaml`의 각 항목은
`examples/` 아래의 파일 하나를 가리키며, `ncm template`이 필요할 때 가져옵니다.

## 사용법

```
ncm template view              모든 항목 나열
ncm template view Wrapper.?    id로 필터링, ?와 *는 와일드카드
ncm template example <api id>  예제 파일 하나를 현재 디렉터리로 가져오기
```

## 두 가지 경로

`NetCraft.ModApi`는 두 개의 네임스페이스를 노출하므로 맞는 것을 고르세요:

| 네임스페이스 | 제공하는 것 |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | 이벤트, `Nc*` 파사드, `Nc*` 핸들. 공개 표면에 커널 타입이 나타나지 않으므로 커널 이름이 바뀌어도 모드를 다시 빌드할 필요가 없습니다. |
| `NetCraft.ModApi.Extension` | `[Inject]`와 `[Mixin]` 애트리뷰트. 규칙이 커널 타입과 메서드를 직접 지정하므로 더 강력하지만 더 취약합니다. |

## 자주 쓰는 호출

모드 프로젝트 안에서 이 패널을 열면, 아래 호출 중 코드가 실제로 사용하는
모든 호출이 `NetCraftTemplate.yaml`을 기준으로 채점됩니다. 멤버가 선언되어
있으면 초록색, 그렇지 않으면 노란색, 타입이 아예 선언되어 있지 않으면
빨간색입니다. 강조된 이름 위에 마우스를 올리면 이유를 볼 수 있습니다.

| 호출 | 하는 일 |
| --- | --- |
| `NcServer.IsAvailable` | 서버가 실행 중이며 캡처되었는지 여부 |
| `NcServer.Broadcast` | 접속 중인 모든 이에게 보내는 시스템 메시지 |
| `NcServer.Execute` | 콘솔로서 명령 실행 |
| `NcWorld.GetBlock` | 블록 하나 읽기, 청크가 언로드되었으면 null |
| `NcWorld.SetBlock` | 블록 하나 쓰기, 전체 업데이트 체인 실행 |
| `NcWorld.BreakBlock` | 플레이어가 하는 것처럼 블록 부수기 |
| `NcPlayer.Name` | 플레이어 이름 |
| `NcPlayer.Health` | 현재 체력 |
| `NcPlayers.Find` | 이름으로 접속 중인 플레이어 찾기 |
| `NcPlayers.Send` | 개인 시스템 메시지 |
| `NcRegistries.FindState` | 네임스페이스 id로 블록 상태 조회 |
| `NcRegistries.FindItem` | 네임스페이스 id로 아이템 조회 |
| `ServerEvents.Tick` | 모든 서버 틱마다 실행 |
| `ServerEvents.PlayerJoin` | 플레이어가 접속을 완료함 |
| `ServerEvents.BlockBroken` | 블록이 실제로 교체됨 |
