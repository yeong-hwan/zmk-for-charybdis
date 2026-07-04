zmk-config for charybdis (4x6)

https://nickcoutsos.github.io/keymap-editor/

```lua
-- 모니터 간 커서 순간이동 (왼쪽부터 1, 2, 3번)
-- Ctrl+Opt+Z → 1번 모니터
-- Ctrl+Opt+X → 2번 모니터
-- Ctrl+Opt+C → 3번 모니터

-- hs.timer는 Lua 참조가 없으면 GC로 수거되어 발화하지 않으므로,
-- 발화 전까지 여기에 보관해서 flash 캔버스가 화면에 남는 것을 방지
local flashTimers = {}

local function deleteCanvasAfter(canvas, delay, fadeOut)
  local timer
  timer = hs.timer.doAfter(delay, function()
    flashTimers[timer] = nil
    canvas:delete(fadeOut)
  end)
  flashTimers[timer] = true
end

local function flashFrame(frame)
  local canvas = hs.canvas.new(frame)
  canvas:appendElements({
    type = "rectangle",
    action = "strokeAndFill",
    fillColor = { white = 1, alpha = 0.18 },
    strokeColor = { red = 1, green = 0, blue = 0, alpha = 0.9 },
    strokeWidth = 6,
    roundedRectRadii = { xRadius = 10, yRadius = 10 },
  })
  canvas:level("overlay")
  canvas:show()
  deleteCanvasAfter(canvas, 0.15, 0.2)
end

local function flashCursor(x, y)
  local size = 50
  local canvas = hs.canvas.new({x = x - size / 2, y = y - size / 2, w = size, h = size})
  canvas:appendElements({
    type = "oval",
    action = "strokeAndFill",
    fillColor = { red = 1, green = 0, blue = 0, alpha = 0.25 },
    strokeColor = { red = 1, green = 0, blue = 0, alpha = 0.95 },
    strokeWidth = 4,
  })
  canvas:level("overlay")
  canvas:show()
  deleteCanvasAfter(canvas, 0.2, 0.25)
end

-- 9분할 2스텝 이동의 세부 모드 상태
-- pendingFrame이 있으면 다음 1~9 입력은 그 프레임 내부를 9분할해서 이동
local pendingFrame = nil
local pendingTimer = nil
local subGridCanvas = nil
local escHotkey = nil

local function clearPending()
  pendingFrame = nil
  if pendingTimer then
    pendingTimer:stop()
    pendingTimer = nil
  end
  if subGridCanvas then
    subGridCanvas:delete(0.15)
    subGridCanvas = nil
  end
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
  subGridCanvas = hs.canvas.new(frame)
  local bw, bh = frame.w / 3, frame.h / 3
  local lineColor = { red = 1, green = 0, blue = 0, alpha = 0.5 }
  subGridCanvas:appendElements({
    type = "rectangle",
    action = "stroke",
    strokeColor = { red = 1, green = 0, blue = 0, alpha = 0.7 },
    strokeWidth = 3,
  })
  for i = 1, 2 do
    subGridCanvas:appendElements({
      type = "segments",
      action = "stroke",
      strokeColor = lineColor,
      strokeWidth = 2,
      coordinates = { { x = bw * i, y = 0 }, { x = bw * i, y = frame.h } },
    }, {
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
      subGridCanvas:appendElements({
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
  subGridCanvas:level("overlay")
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

