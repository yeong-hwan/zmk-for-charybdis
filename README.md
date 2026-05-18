zmk-config for charybdis (4x6)

https://nickcoutsos.github.io/keymap-editor/

```lua
-- 모니터 간 커서 순간이동 (왼쪽부터 1, 2, 3번)
-- Ctrl+Opt+Z → 1번 모니터
-- Ctrl+Opt+X → 2번 모니터
-- Ctrl+Opt+C → 3번 모니터

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
  hs.timer.doAfter(0.15, function()
    canvas:delete(0.2)
  end)
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
  hs.timer.doAfter(0.2, function()
    canvas:delete(0.25)
  end)
end

local function moveCursorToScreen(screen)
  if not screen then return end
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

-- 현재 화면 9분할 블럭 중앙으로 커서 이동
-- Ctrl+Opt+1~9 (numpad 배치: 1=좌하단, 9=우상단)
local function moveCursorToBlock(row, col)
  local screen = hs.mouse.getCurrentScreen() or hs.screen.mainScreen()
  local f = screen:frame()
  local bw = f.w / 3
  local bh = f.h / 3
  local cx = f.x + bw * (col - 0.5)
  local cy = f.y + bh * (row - 0.5)
  hs.mouse.absolutePosition({x = cx, y = cy})
  flashFrame({
    x = f.x + bw * (col - 1),
    y = f.y + bh * (row - 1),
    w = bw,
    h = bh,
  })
  flashCursor(cx, cy)
end

for i = 1, 9 do
  local row = 3 - math.floor((i - 1) / 3)
  local col = ((i - 1) % 3) + 1
  hs.hotkey.bind({"ctrl", "alt"}, tostring(i), function()
    moveCursorToBlock(row, col)
  end)
end

hs.alert.show("Hammerspoon config loaded")

```
