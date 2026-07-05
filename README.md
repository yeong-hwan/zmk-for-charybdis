zmk-config for charybdis (4x6)

> 🤖 **AI 에이전트**: 이 저장소에서 작업(키맵 수정, 감도 조절, 빌드, 플래시, ZMK 포크 패치)을
> 시작하기 전에 반드시 [AGENT.md](./AGENT.md)를 먼저 읽을 것. 문서의 레시피와 수칙에 따라
> 자동화 가능한 부분은 직접 수행하고, 물리 작업(USB 연결, 리셋 더블탭)만 사용자에게 안내한다.

https://nickcoutsos.github.io/keymap-editor/

```lua
-- 모니터 간 커서 순간이동 (왼쪽부터 1, 2, 3번)
-- Ctrl+Opt+Z → 1번 모니터
-- Ctrl+Opt+X → 2번 모니터
-- Ctrl+Opt+C → 3번 모니터

-- hs.canvas는 new→show→delete 사이클마다 화면 버퍼(CALayer 백킹스토어)를
-- 해제하지 않고 누수하므로(풀스크린 1회당 수십 MB), 캔버스는 시작 시
-- 한 번만 만들어 두고 show/hide로 재사용한다
local function makeFlash(element, delay, fadeOut)
  local canvas = hs.canvas.new({ x = 0, y = 0, w = 1, h = 1 })
  canvas:appendElements(element)
  canvas:level("overlay")
  local hideTimer = nil
  return function(frame)
    canvas:frame(frame)
    canvas:show()
    if hideTimer then hideTimer:stop() end
    hideTimer = hs.timer.doAfter(delay, function()
      canvas:hide(fadeOut)
    end)
  end
end

local flashFrame = makeFlash({
  type = "rectangle",
  action = "strokeAndFill",
  fillColor = { white = 1, alpha = 0.18 },
  strokeColor = { red = 1, green = 0, blue = 0, alpha = 0.9 },
  strokeWidth = 6,
  roundedRectRadii = { xRadius = 10, yRadius = 10 },
}, 0.15, 0.2)

local flashCursorFrame = makeFlash({
  type = "oval",
  action = "strokeAndFill",
  fillColor = { red = 1, green = 0, blue = 0, alpha = 0.25 },
  strokeColor = { red = 1, green = 0, blue = 0, alpha = 0.95 },
  strokeWidth = 4,
}, 0.2, 0.25)

local function flashCursor(x, y)
  local size = 50
  flashCursorFrame({ x = x - size / 2, y = y - size / 2, w = size, h = size })
end

-- 9분할 2스텝 이동의 세부 모드 상태
-- pendingFrame이 있으면 다음 1~9 입력은 그 프레임 내부를 9분할해서 이동
local pendingFrame = nil
local pendingTimer = nil
local subGridCanvas = hs.canvas.new({ x = 0, y = 0, w = 1, h = 1 }):level("overlay")
local escHotkey = nil

local function clearPending()
  pendingFrame = nil
  if pendingTimer then
    pendingTimer:stop()
    pendingTimer = nil
  end
  subGridCanvas:hide(0.15)
  if escHotkey then
    escHotkey:disable()
  end
end

escHotkey = hs.hotkey.new({}, "escape", clearPending)

local function moveCursorToScreen(screen)
  if not screen then return end
  clearPending()
  local f = screen:frame()
  local cx, cy = f.x + f.w / 2, f.y + f.h / 2
  hs.mouse.absolutePosition({x = cx, y = cy})
  flashFrame(f)
  flashCursor(cx, cy)
end

local function screensLeftToRight()
  local screens = hs.screen.allScreens()
  table.sort(screens, function(a, b) return a:frame().x < b:frame().x end)
  return screens
end

local function bindScreen(key, idx)
  hs.hotkey.bind({"ctrl", "alt"}, key, function()
    moveCursorToScreen(screensLeftToRight()[idx])
  end)
end

bindScreen("z", 1)
bindScreen("x", 2)
bindScreen("c", 3)

-- 현재 화면 9분할 블럭 중앙으로 커서 이동 (2스텝)
-- Ctrl+Opt+1~9 (numpad 배치: 1=좌하단, 9=우상단)
-- 첫 입력: 화면 9분할 블럭으로 이동 + 1.5초간 세부 모드 진입
-- 세부 모드 중 Ctrl+Opt+1~9: 그 블럭을 다시 9분할해서 이동
-- 타임아웃 / Esc / 모니터 이동 키(z/x/c) → 세부 모드 종료

local SUBGRID_TIMEOUT = 1.5

local function gridCellFrame(f, row, col)
  local bw, bh = f.w / 3, f.h / 3
  return {
    x = f.x + bw * (col - 1),
    y = f.y + bh * (row - 1),
    w = bw,
    h = bh,
  }
end

local function moveCursorToCell(f, row, col)
  local cell = gridCellFrame(f, row, col)
  local cx, cy = cell.x + cell.w / 2, cell.y + cell.h / 2
  hs.mouse.absolutePosition({x = cx, y = cy})
  flashFrame(cell)
  flashCursor(cx, cy)
  return cell
end

-- 세부 모드 동안 블럭 내부에 3×3 그리드와 numpad 숫자 라벨 표시
local function showSubGrid(frame)
  local els = {}
  local bw, bh = frame.w / 3, frame.h / 3
  local lineColor = { red = 1, green = 0, blue = 0, alpha = 0.5 }
  table.insert(els, {
    type = "rectangle",
    action = "stroke",
    strokeColor = { red = 1, green = 0, blue = 0, alpha = 0.7 },
    strokeWidth = 3,
  })
  for i = 1, 2 do
    table.insert(els, {
      type = "segments",
      action = "stroke",
      strokeColor = lineColor,
      strokeWidth = 2,
      coordinates = { { x = bw * i, y = 0 }, { x = bw * i, y = frame.h } },
    })
    table.insert(els, {
      type = "segments",
      action = "stroke",
      strokeColor = lineColor,
      strokeWidth = 2,
      coordinates = { { x = 0, y = bh * i }, { x = frame.w, y = bh * i } },
    })
  end
  local textSize = math.min(bw, bh) * 0.3
  for row = 1, 3 do
    for col = 1, 3 do
      local num = (3 - row) * 3 + col
      table.insert(els, {
        type = "text",
        text = tostring(num),
        textSize = textSize,
        textAlignment = "center",
        textColor = { red = 1, green = 0, blue = 0, alpha = 0.7 },
        frame = {
          x = bw * (col - 1),
          y = bh * (row - 1) + (bh - textSize * 1.3) / 2,
          w = bw,
          h = textSize * 1.3,
        },
      })
    end
  end
  subGridCanvas:replaceElements(table.unpack(els))
  subGridCanvas:frame(frame)
  subGridCanvas:show()
end

local function handleGridKey(row, col)
  if pendingFrame then
    local base = pendingFrame
    clearPending()
    moveCursorToCell(base, row, col)
    return
  end
  local screen = hs.mouse.getCurrentScreen() or hs.screen.mainScreen()
  local cell = moveCursorToCell(screen:frame(), row, col)
  pendingFrame = cell
  showSubGrid(cell)
  escHotkey:enable()
  pendingTimer = hs.timer.doAfter(SUBGRID_TIMEOUT, clearPending)
end

for i = 1, 9 do
  local row = 3 - math.floor((i - 1) / 3)
  local col = ((i - 1) % 3) + 1
  hs.hotkey.bind({"ctrl", "alt"}, tostring(i), function()
    handleGridKey(row, col)
  end)
end

hs.alert.show("Hammerspoon config loaded")
```
