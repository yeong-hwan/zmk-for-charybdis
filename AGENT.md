# AGENT.md — AI 에이전트 작업 가이드

이 문서는 AI 에이전트(Claude Code 등)가 이 저장소에서 펌웨어 수정·빌드·플래시 작업을
사용자 개입 최소화로 수행하기 위한 가이드다. 작업 전 반드시 끝까지 읽을 것.

## 1. 저장소 개요

- **하드웨어**: Charybdis 4x6 스플릿 무선 키보드. 컨트롤러는 양쪽 모두 nice!nano v2
  (nRF52840), **오른쪽 반쪽에 PMW3610 트랙볼 센서**. RGB 없음(비활성).
- **펌웨어**: ZMK — 단, 공식 ZMK가 아니라 **구형(2021년경) 커스텀 포크** 기반.
  - 소스: `yeong-hwan/zmk`의 `feature-test` 브랜치 (upstream: `DoctorWangWang/zmk`)
  - `config/west.yml`의 `url-base`가 포크를 가리킨다. revision이 브랜치명이므로
    포크에 커밋을 푸시하면 다음 빌드에 자동 반영된다.
  - **주의**: 공식 ZMK 문서의 pointing/input-processor 문법은 이 포크에 없다.
    트랙볼은 포크 고유의 `trackball-bindings`, `&tmv*`, `&tsl*` 동작을 쓴다.
- **호스트**: 사용자는 macOS(Apple Silicon).

## 2. 파일 구조

| 경로 | 내용 |
|------|------|
| `config/charybdis.keymap` | 키맵(4개 레이어) + 동작 파라미터 오버라이드 |
| `config/west.yml` | ZMK 소스 저장소/브랜치 지정 |
| `config/charybdis.conf` | BT 파라미터 등 공통 설정 |
| `config/boards/shields/charybdis/` | 셰일드 정의 — 매트릭스, 트랙볼 SPI, PMW3610 설정(`charybdis_right.conf`) |
| `.github/workflows/build.yml` | 푸시/수동 트리거 시 펌웨어 빌드 (~1분 30초) |

키맵 요점: 레이어0에서 트랙볼은 포인터 이동(`&tmv_coarse`), 레이어1·2에서는
스크롤(`&tsl`). MB1은 레이어0 왼쪽 엄지, MB2는 레이어1(mo 1 홀드) 왼쪽 엄지.

## 3. 자주 하는 작업 레시피

### 3-1. 스크롤 감도 조절
`config/charybdis.keymap` 상단의 오버라이드 수정:
```dts
&tsl {
    scale_factor = <4>;   // 낮을수록 민감. 포크 기본값 10, 현재 4
};
```
정수 나눗셈이라 `scale_factor` 미만의 미세 움직임은 버려진다. 부드러운(고해상도)
스크롤은 이 포크가 지원하지 않음 — 그건 macOS 쪽 LinearMouse로 보간한다.

### 3-2. 포인터 감도 조절
- 레이어별: keymap의 `trackball-bindings`를 `&tmv_coarse`(÷1) / `&tmv`(÷2) / `&tmv_fine`(÷4)로 교체
- 전역 CPI: `config/boards/shields/charybdis/charybdis_right.conf`의
  `CONFIG_PMW3610_CPI`(현재 400), `CONFIG_PMW3610_CPI_DIVIDOR`(현재 2)
- 동작 정의 원본: 포크의 `app/dts/behaviors/trackball.dtsi`

### 3-3. ZMK 소스(포크) 패치
1. `yeong-hwan/zmk`를 클론: `git clone --single-branch --branch feature-test https://github.com/yeong-hwan/zmk`
2. 수정 → 커밋 → `feature-test`에 푸시 (계정 주의사항은 §5)
3. 이 저장소의 빌드를 다시 트리거: `gh workflow run build.yml --repo yeong-hwan/zmk-for-charybdis`
   (west.yml이 브랜치를 가리키므로 이 저장소에 커밋이 없어도 최신 포크가 반영됨)

### 3-4. 빌드 확인 및 펌웨어 다운로드
```bash
gh run list --repo yeong-hwan/zmk-for-charybdis --limit 3
gh run watch <run-id> --repo yeong-hwan/zmk-for-charybdis --exit-status
gh run download <run-id> --repo yeong-hwan/zmk-for-charybdis --name firmware --dir <dir>
```
아티팩트 `firmware`에 `charybdis_left…uf2`, `charybdis_right…uf2`, `settings_reset…uf2`가 들어있다.

### 3-5. 플래시 (macOS) — 중요한 함정 있음
사용자가 해야 하는 물리 작업은 두 가지뿐: **① 해당 반쪽 USB 연결 ② 리셋 버튼 빠르게 더블탭**.
나머지는 에이전트가 자동화한다. 반드시 아래 사실을 반영할 것:

- **반쪽 뒤바뀜 사고 주의 (실제 발생)**: 잘못된 반쪽에 반대편 uf2를 구우면 키가
  거울상처럼 뒤집혀 동작한다(우측 셰일드의 `col-offset = <6>` 때문). 플래시 안내 시
  "좌측/우측" 단어만 쓰지 말고 반드시 물리적 기준으로 확인시킬 것:
  **트랙볼이 있는 쪽 = 우측(`charybdis_right`), 없는 쪽 = 좌측(`charybdis_left`)**.
  같은 보드가 연속으로 두 번 부트로더에 들어와 두 번째 uf2를 덮어쓰는 경우도 있으니,
  두 반쪽을 연달아 플래시할 때는 보드를 실제로 바꿔 꽂았는지 짚고 넘어갈 것.

- 부트로더 진입 시 USB에 `nice!nano` 장치 + `NICENANO` 디스크가 잡히지만,
  **이 Mac은 볼륨을 자동 마운트하지 않는다** → `diskutil mount`를 직접 해야 한다.
- 복사 마지막에 `fcopyfile failed: Input/output error`가 나는 것은 **정상**
  (보드가 펌웨어 수신 완료 후 스스로 재부팅하며 드라이브가 끊김).
- **복사가 불완전하게 끝나 부트로더에 남는 경우가 잦다** (이 Mac에서 3회 중 2회 발생).
  성공 판정은 cp 종료코드가 아니라 재열거 확인으로 한다: `system_profiler SPUSBDataType`에
  `ZMK Project`가 보이면 성공, 여전히 `nice!nano`면 재마운트 후 다시 복사.
- 감시+마운트+복사+검증+재시도 자동화(백그라운드로 실행해두고 사용자에게 더블탭 안내):
```bash
for attempt in 1 2 3; do
  until [ -d /Volumes/NICENANO ]; do
    disk=$(diskutil list external physical 2>/dev/null | awk '/NICENANO/ {print $NF}' | head -1)
    [ -n "$disk" ] && diskutil mount "$disk" >/dev/null 2>&1
    sleep 1
  done
  cp "<펌웨어>.uf2" /Volumes/NICENANO/ 2>/dev/null   # 끝에 I/O error 나는 것은 정상
  sleep 6
  system_profiler SPUSBDataType 2>/dev/null | grep -q "ZMK Project" && { echo "FLASH_OK"; exit 0; }
  echo "attempt $attempt: still in bootloader, retrying"
done
echo "FLASH_FAILED after 3 attempts"
```
- keymap/동작 변경은 양쪽 모두 플래시가 원칙. 설정 파티션은 유지되므로
  BT 페어링은 보통 그대로 살아있다. 좌우 연결이 깨졌을 때만 `settings_reset`을
  양쪽에 굽고 재페어링.

## 4. 히스토리 / 알려진 이슈

- **클릭 시 커서 튐 (2026-07 수정됨)**: 포크의 전역 `mouse_report` 버퍼가 전송 후
  이동 델타를 지우지 않아, 버튼 press/release 리포트에 마지막 트랙볼 델타가 실려
  재전송되던 버그. `app/src/mouse/key_listener.c`에서 버튼 리포트 전송 전
  `zmk_hid_mouse_movement_set(0,0)` / `zmk_hid_mouse_scroll_set(0,0)` 호출로 수정
  (포크 커밋 `14e8f5e`). 유사 증상 재발 시 이 지점부터 의심할 것.
- 부트로더는 UF2 0.6.0 (구형). 볼륨명 `NICENANO`.
- 이 포크는 고해상도 스크롤(HID Resolution Multiplier) 미지원.
- 장기 과제: 공식 ZMK + 커뮤니티 PMW3610 드라이버로 이전하면 위 버그·제약이
  없어지지만 keymap 마이그레이션 필요.

## 5. GitHub 계정 주의사항

- `gh`에 두 계정이 로그인돼 있다: `backend-yeonghwan-chang`(회사, 평소 활성),
  `yeong-hwan`(개인). **이 저장소와 zmk 포크는 개인 계정 소유.**
- 푸시 전: `gh auth switch --user yeong-hwan`
- macOS 키체인이 회사 계정 자격증명을 먼저 내밀어 push가 403으로 실패한다.
  반드시 gh를 credential helper로 강제할 것:
  ```bash
  git -c credential.helper= -c credential.helper='!gh auth git-credential' push origin <branch>
  ```
- **작업이 끝나면 반드시 복귀**: `gh auth switch --user backend-yeonghwan-chang`

## 6. 사용자 안내 수칙

- 물리 작업(USB 연결, 리셋 더블탭)은 시점을 명확히 짚어 안내하고, 그 외 전 과정
  (빌드 감시, 다운로드, 마운트, 복사, 검증)은 에이전트가 자동으로 수행한다.
- 플래시 성공/실패는 반드시 재열거 확인 후 보고한다. 추측으로 성공을 선언하지 말 것.
- 검증 방법까지 안내할 것 — 예: 클릭 튐은 "볼을 한 방향으로 굴리고 멈춘 뒤 클릭"으로 확인.
