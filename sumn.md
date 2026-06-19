# Matcha LuaVM — Knowledge Base
> Compiled from session notes and tested scripts. Last updated this session.

---

## What is Matcha

Matcha is an external executor/cheat tool for Roblox that emulates an internal executor through a VM layer. It runs Lua scripts externally but gives access to game internals like memory reading, input simulation, and drawing. It has its own UI framework for building tabs and menus that attach to Matcha's interface.

---

## UNC Test Results (92/98 passing)

### PASSING
- All standard Lua globals: `pcall`, `pairs`, `ipairs`, `setmetatable` etc
- `identifyexecutor`, `getscripts`, `getscripthash`, `getscriptbytecode`
- `base64encode` / `base64decode`
- `WorldToScreen`
- `setclipboard`
- All input functions: `keypress`, `keyrelease`, `ismouse1pressed`, `iskeypressed` etc
- `getbase`, `memory_read`, `memory_write` (exist, unsafe mode required to use)
- `run_secure`, `setfflag`, `getfflag`, `require`, `decompile` (exist)
- `game:GetService` — Players, UserInputService, HttpService all work
- `HttpService:JSONEncode`, `JSONDecode`, `GenerateGUID`
- `game.PlaceId`, `GameId`, `JobId`
- `Players.LocalPlayer`, `GetPlayers`, `GetMouse`
- `Character`, `Humanoid`, `HumanoidRootPart`
- All Instance methods: `FindFirstChild`, `FindFirstChildOfClass`, `IsA`, `GetFullName`, `GetChildren`, `GetDescendants`, `IsDescendantOf`, `FindFirstChildWhichIsA`, `WaitForChild`, `GetAttribute`, `SetAttribute`
- `Instance.Address` (memory address as number)
- `Vector3.new`, `Vector2.new`
- `Color3.new`, `fromRGB`, `fromHSV`, `fromHex`
- `Drawing.new` (Square, Line, Circle, Text, Triangle)
- Drawing properties: Color, Transparency, Visible, Position, ZIndex etc
- `Drawing.Fonts`: UI, System, SystemBold, Minecraft, Monospace, Pixel, Fortnite
- `workspace.CurrentCamera` (ViewportSize, FieldOfView, Position)
- `task.spawn`, `task.wait`
- `spawn`, `wait`
- `os.clock`
- `writefile`, `makefolder` (file system access confirmed working)
- `listfiles` (may or may not work depending on Matcha version)

### FAILING / BROKEN
- `getgetname` — nil, not implemented despite being in docs
- `CFrame.new` — nil, CFrame global does not exist
- `CFrame.Angles` — nil, same reason
- `GetAttribute`/`SetAttribute` round-trip — values not preserved correctly
- `Humanoid:GetState()` — fails on other players' humanoids
- `Humanoid:GetPlayingAnimationTracks()` — fails entirely
- `loadstring` — exists but return value is non-standard
- Drawing "Text" type — exists but known Matcha bug, doesn't render
- `Player.CharacterAdded` — not a connectable event in Matcha
- `goto` keyword — not supported (Lua 5.1 base, no goto)

---

## CFrame — The Big Problem and Workarounds

`CFrame.new` and `CFrame.Angles` are both nil. You cannot construct CFrames from scratch.

### What you CAN do

```lua
-- Reading CFrame properties works fine
local cf    = hrp.CFrame          -- CFrame object
local pos   = hrp.Position        -- Vector3
local look  = cf.LookVector       -- Vector3
local up    = cf.UpVector         -- Vector3
local right = cf.RightVector      -- Vector3

-- Derive positions without constructing CFrames
local behind = hrp.Position + hrp.CFrame.LookVector * distance
local above  = hrp.Position + Vector3.new(0, offset, 0)
local below  = hrp.Position - Vector3.new(0, offset, 0)

-- Lerp works on existing CFrame objects
local lerped = cf:Lerp(otherCf, 0.5)

-- ToEulerAnglesXYZ works on existing CFrames
local rx, ry, rz = cf:ToEulerAnglesXYZ()

-- Motor6D.C0 can be read but not meaningfully written
```

### Memory-based CFrame replacement (THE big workaround)

Since CFrame.new is nil, the only way to actually set rotation and position is through memory writes. This is a major capability — it's essentially the CFrame replacement for Matcha.

```lua
-- Fetch offsets from external JSON (game-version-specific)
local HttpService = game:GetService("HttpService")
local response = game:HttpGet("https://offsets.ntgetwritewatch.workers.dev/offsets.json")
local offsets = HttpService:JSONDecode(response)

local hrp = game.Players.LocalPlayer.Character.HumanoidRootPart
local primitive = memory_read("uintptr", hrp.Address + offsets.Primitive)

-- Convert yaw degrees to rotation matrix components
local yawDegrees = 90
local yaw = yawDegrees * math.pi / 180
local fx = math.cos(yaw)
local fz = math.sin(yaw)

-- Build 3x3 rotation matrix
local r00=-fz  local r01=0  local r02=-fx
local r10=0    local r11=1  local r12=0
local r20=fx   local r21=0  local r22=-fz

local cf  = offsets.CFrame
local vel = offsets.Velocity

-- Read current position
local px = memory_read("float", primitive + offsets.Position)
local py = memory_read("float", primitive + offsets.Position + 0x4)
local pz = memory_read("float", primitive + offsets.Position + 0x8)

-- Write rotation matrix + position + zero velocity
for i = 1, 10000 do
    memory_write("float", primitive + cf + 0x00, r00)
    memory_write("float", primitive + cf + 0x04, r01)
    memory_write("float", primitive + cf + 0x08, r02)
    memory_write("float", primitive + cf + 0x0C, r10)
    memory_write("float", primitive + cf + 0x10, r11)
    memory_write("float", primitive + cf + 0x14, r12)
    memory_write("float", primitive + cf + 0x18, r20)
    memory_write("float", primitive + cf + 0x1C, r21)
    memory_write("float", primitive + cf + 0x20, r22)
    memory_write("float", primitive + cf + 0x24, px)
    memory_write("float", primitive + cf + 0x28, py)
    memory_write("float", primitive + cf + 0x2C, pz)
    memory_write("float", primitive + vel + 0x0, 0)
    memory_write("float", primitive + vel + 0x4, 0)
    memory_write("float", primitive + vel + 0x8, 0)
end
```

---

## Memory Reading

Matcha exposes `memory_read` and `memory_write` in unsafe mode.

```lua
-- Read types
memory_read("uintptr",  addr)          -- pointer / address
memory_read("uintptr_t", addr)         -- same thing
memory_read("float",    addr)          -- float32
memory_read("string",   addr)          -- string pointer
memory_read("byte",     addr)          -- single byte

-- Write
memory_write("float", addr, value)

-- Always pcall memory reads — they throw on bad addresses
local ok, val = pcall(memory_read, "float", addr)
if ok and val then ... end

-- Address guard — skip obviously invalid pointers
if not addr or addr <= 4096 then return nil end
```

### Confirmed working offsets (version-6776addb8fbc4d17)

```lua
local VISIBLE_OFFSET = 1461   -- GuiObject.Visible (byte)
local SOUNDID_OFFSET = 224    -- Sound.SoundId (string ptr)
local VALUE_OFFSET   = 208    -- Misc.Value (float)

local INST = {
    ClassDescriptor = 24,
    ClassName       = 8,
    Name            = 176,
    ChildrenStart   = 120,   -- was 112 in older versions
    ChildNode       = 8,
}

local ANIM = {
    ActiveAnimations = 2120, -- Animator.ActiveAnimations (was 2152)
    Animation        = 208,
    AnimationId      = 208,
    NodeNext         = 16,
}

local HUM = {
    Health    = 404,
    MaxHealth = 436,
}

local CAM = {
    ViewportSize = 744,
}

local GUI = {
    AbsolutePosition = 272,
    AbsoluteSize     = 280,
}
```

### Reading animation tracks via memory (replaces GetPlayingAnimationTracks)

Since `GetPlayingAnimationTracks()` fails on remote players, walk the linked list manually:

```lua
local function readStr(addr)
    if not addr or addr <= 4096 then return nil end
    local ok, v = pcall(memory_read, "string", addr)
    return ok and v or nil
end

local function readPtr(addr)
    if not addr or addr <= 4096 then return nil end
    local ok, v = pcall(memory_read, "uintptr_t", addr)
    return (ok and v and v > 4096) and v or nil
end

local function getClass(addr)
    if not addr or addr <= 4096 then return nil end
    local desc = readPtr(addr + INST.ClassDescriptor)
    if not desc then return nil end
    local namePtr = readPtr(desc + INST.ClassName)
    return namePtr and readStr(namePtr) or nil
end

local function getAnimator(char)
    local hum  = char:FindFirstChild("Humanoid")
    if not hum then return nil end
    local anim = hum:FindFirstChild("Animator")
    return anim and tonumber(anim.Address) or nil
end

local function getCurrentAnimFromChar(char, mode)
    local animator = getAnimator(char)
    if not animator then return nil end
    local head = readPtr(animator + ANIM.ActiveAnimations)
    if not head then return nil end
    local first = readPtr(head)
    if not first or first == head then return nil end
    local curr = first
    local maxIter, iter = 64, 0
    while curr and curr ~= 0 and curr ~= head and iter < maxIter do
        iter = iter + 1
        local track = readPtr(curr + ANIM.NodeNext)
        if track and getClass(track) == "AnimationTrack" then
            local anim = readPtr(track + ANIM.Animation)
            if anim and getClass(anim) == "Animation" then
                local idPtr = readPtr(anim + ANIM.AnimationId)
                local idStr = readStr(idPtr)
                if idStr and idStr ~= "N/A" then
                    for id in pairs(mode) do
                        if string.find(idStr, id, 1, true) then return id end
                    end
                end
            end
        end
        curr = readPtr(curr)
    end
    return nil
end
```

---

## Getting Ping

`GetNetworkPing()` exists but is unreliable in Matcha. Better alternative via Stats service:

```lua
local function GetPing()
    local ok, v = pcall(function()
        return game:GetService("Stats").Network.ServerStatsItem["Data Ping"].Value
    end)
    return (ok and tonumber(v)) or 0
end
```

---

## Loop Patterns

No RunService in Matcha. Use `task.spawn` + `task.wait` loops.

```lua
-- Standard polling loop
local running = true
task.spawn(function()
    while running do
        task.wait(0.05)  -- 20hz
        -- your code
    end
end)

-- Stop it
running = false

-- task.wait() with no argument = one frame (~0.016s at 60fps)
-- os.clock() for timing
-- No goto — use nested ifs instead
-- No Enum.HumanoidStateType — enums not available
-- No RunService timeout — loops run forever until running = false
```

---

## Input System

VK codes (Windows virtual key codes):

```lua
-- Mouse
0x01  -- LMB
0x02  -- RMB
0x04  -- MMB

-- Letters (0x41-0x5A = A-Z)
0x41=A  0x42=B  0x43=C  0x44=D  0x45=E
0x46=F  0x47=G  0x48=H  0x49=I  0x4A=J
0x4B=K  0x4C=L  0x4D=M  0x4E=N  0x4F=O
0x50=P  0x51=Q  0x52=R  0x53=S  0x54=T
0x55=U  0x56=V  0x57=W  0x58=X  0x59=Y  0x5A=Z

-- Numbers (0x30-0x39 = 0-9)
0x30=0  0x31=1  0x32=2  0x33=3  0x34=4
0x35=5  0x36=6  0x37=7  0x38=8  0x39=9

-- Function keys
0x70=F1  0x71=F2  0x72=F3  0x73=F4

-- Input functions
iskeypressed(vk)    -- returns bool
keypress(vk)
keyrelease(vk)
mouse1press()
mouse1release()
mouse1click()
ismouse1pressed()
```

`input.KeyCode` in `UserInputService.InputBegan` returns raw VK integer. No `Enum.KeyCode` support — use raw VK codes.

---

## Drawing API

```lua
local line = Drawing.new("Line")
line.Thickness = 1
line.Color = Color3.fromRGB(255, 0, 0)
line.Visible = true
line.From = Vector2.new(0, 0)
line.To = Vector2.new(100, 100)

local text = Drawing.new("Text")
text.Font = Drawing.Fonts.System  -- UI, System, SystemBold, Minecraft, Monospace, Pixel, Fortnite
text.Size = 16
text.Text = "hello"
text.Color = Color3.fromRGB(255, 255, 255)
text.Outline = true
text.Center = true
text.Visible = true
text.Position = Vector2.new(100, 100)

-- NOTE: Drawing "Text" type has a known Matcha bug and sometimes doesn't render
-- notify() is more reliable for on-screen text

-- Remove a drawing
drawing:Remove()

-- WorldToScreen — converts 3D world position to 2D screen position
local screenPos, onScreen = WorldToScreen(worldPos)
-- screenPos = Vector2, onScreen = bool
-- Always pcall this — can throw if camera not ready
local ok, screenPos, onScreen = pcall(WorldToScreen, position)
```

---

## Color3 Gotcha

`Color3.new()` takes 0-1 values, NOT 0-255.
`Color3.fromRGB()` takes 0-255.
`BrickColor.Color` returns a 0-1 Color3 already — do NOT multiply by 255.

```lua
Color3.new(1, 1, 1)           -- white, correct
Color3.new(255, 255, 255)     -- WRONG, will blow out to white clipped
Color3.fromRGB(255, 255, 255) -- white, correct
torso.BrickColor.Color        -- already 0-1, use directly
```

---

## What Replicates to Client vs What Doesn't

### DOES replicate (readable on other players)
- Character model and all children (Parts, Attachments, Motor6Ds)
- Humanoid (Health, MaxHealth)
- HumanoidRootPart.Position, Velocity
- StringValue, NumberValue, BoolValue, IntValue contents
- Instance names and class names
- Scripts listed as children (name/class visible, not content)
- AnimVal (StringValue on character) — game-specific, very useful

### DOES NOT replicate
- Animator children (always 0, AnimationTracks not visible)
- `Humanoid:GetState()` — fails on remote players
- `Humanoid:GetPlayingAnimationTracks()` — fails entirely
- CFrame construction (CFrame global is nil)

---

## Safe Coding Patterns

### Always pcall remote instance reads
```lua
local ok, val = pcall(function() return instance.SomeProperty end)
if ok and val then ... end

-- pcall on IsA (can return weird values in Matcha)
local ok, result = pcall(function() return instance:IsA("BasePart") end)
if ok and result == true then ... end  -- explicit true check
```

### Safe chained FindFirstChild
```lua
local function SafeFind(parent, ...)
    if not parent then return nil end
    for _, name in ipairs({...}) do
        parent = parent:FindFirstChild(name)
        if not parent then return nil end
    end
    return parent
end

-- Usage
local healthBar = SafeFind(LocalPlayer, "PlayerGui", "Main", "Health", "Bar")
```

### Squared distance (skip math.sqrt in tight loops)
```lua
local function distSq(a, b)
    local dx = b.X - a.X
    local dy = b.Y - a.Y
    local dz = b.Z - a.Z
    return dx*dx + dy*dy + dz*dz
end

if distSq(posA, posB) < (RANGE * RANGE) then
    -- in range
end
```

### Defer LocalPlayer reads
```lua
-- WRONG — LocalPlayer can be nil at script start, causes crash
local LocalPlayer = Players.LocalPlayer  -- at top level

-- CORRECT — fetch inside functions
local function doThing()
    local LocalPlayer = Players.LocalPlayer
    if not LocalPlayer then return end
    ...
end
```

### Use next instead of ipairs for non-contiguous tables
```lua
-- If index continuity isn't guaranteed
for k, v in next, myTable do ... end
```

---

## Matcha UI System

Full menu framework via the global `UI`. Tabs, sections, and widgets attach to Matcha's built-in interface.

```lua
UI.AddTab("Tab Name", function(tab)
    local sec = tab:Section("Section Name", "Left")   -- or "Right"
    local secTabbed = tab:Section("Name", "Left", {"Page 1", "Page 2"}, maxHeight)

    -- page detection
    if secTabbed.page == 0 then ... end
    if secTabbed.page == 1 then ... end

    sec:Toggle("id", "Label", defaultBool, callback)
    sec:Keybind("id", 0x46, "toggle")   -- or "hold", "always", "click"
    sec:SliderInt("id", "Label", min, max, default, callback)
    sec:SliderFloat("id", "Label", min, max, default, "%.1f", callback)
    sec:Combo("id", "Label", {"Option1", "Option2"}, defaultIndex, callback)
    sec:Button("Label", callback)
    sec:Button("Label", width, height, callback)
    sec:InputText("id", "Label", default, callback)
    sec:ColorPicker("id", r, g, b, a, callback)
    sec:ColorPicker2("id1", {r,g,b,a}, "id2", {r,g,b,a}, callback)
    sec:Text("some text")
    sec:Tip("tooltip for previous widget")
    sec:Spacing()
end)

UI.RemoveTab("Tab Name")
UI.GetValue("id")       -- read any widget value
UI.SetValue("id", val)  -- write any widget value
```

### Keybind widget methods
```lua
local kb = sec:Keybind("id", 0x46, "hold")
kb:IsEnabled()        -- is it active right now
kb:GetKey()           -- VK code
kb:SetKey(vk)
kb:GetKeyName()       -- "f", "lmb", etc
kb:GetType()          -- "toggle", "hold", "always", "click"
kb:SetType(str)
kb:AddToHotkey("Label", "toggle_id")   -- show in hotkey overlay when toggle is ON
kb:RemoveFromHotkey()
```

### Combo widget methods
```lua
local c = sec:Combo("id", "Label", items)
c:Add("item")
c:Remove("item")
c:Clear()
c:GetItems()
c:GetText()
c:SetValue(index)
```

### Widget value types
| Widget | Value type |
|--------|-----------|
| Toggle | bool |
| SliderInt | int |
| SliderFloat | float |
| Combo | int (0-based) |
| InputText | string |
| ColorPicker | r, g, b, a |
| Keybind | bool |

---

## Notification

```lua
notify("message", "title", durationSeconds)
```

More reliable than Drawing.new("Text") which has a known Matcha bug.

---

## File System

```lua
writefile("filename.txt", content)     -- saves to C:\matcha\workspace\
makefolder("foldername")               -- creates folder in workspace
listfiles("foldername")                -- may or may not work depending on version
setclipboard(text)                     -- copies to clipboard
```

---

## loadstring Hook Pattern

Use this to intercept and dump obfuscated scripts at runtime:

```lua
local old_loadstring = loadstring
loadstring = function(str, ...)
    if type(str) == "string" and #str > 0 then
        setclipboard(str)
        notify("dumped to clipboard (" .. #str .. " chars)", "loadstring hook", 4)
    end
    return old_loadstring(str, ...)
end
```

Do NOT hook `string.char` — it gets called for everything internally and will crash Matcha with thousands of prints.

---

## Workspace Dump Pattern

For understanding a new game's structure before scripting it:

```lua
local lines = {}
local function listDescendants(instance, indent)
    if not instance then return end
    indent = indent or 0
    table.insert(lines, string.rep("  ", indent) .. instance.Name .. " [" .. instance.ClassName .. "]")
    for _, child in ipairs(instance:GetChildren()) do
        listDescendants(child, indent + 1)
    end
end
listDescendants(game.Workspace)
setclipboard(table.concat(lines, "\n"))
notify("copied " .. #lines .. " entries", "dump", 3)
```

Also scan `ReplicatedStorage` for RemoteEvents — they almost never live in Workspace directly even if they appear to from the game's script names.

---

## AnimVal Pattern (game-specific)

Some games store the current animation state in a `StringValue` named `AnimVal` on the character. Watching this is simpler than memory walking when available:

```lua
local function watchAnimVal(player)
    local char = player.Character
    if not char then return end
    local animVal = char:FindFirstChild("AnimVal")
    if not animVal then return end
    local last = ""
    task.spawn(function()
        while running do
            task.wait(0.05)
            local ok, val = pcall(function() return animVal.Value end)
            if ok and type(val) == "string" and val ~= last then
                last = val
                print(player.Name .. ": " .. val)
            end
        end
    end)
end
```

---

## Humanoid Health via Memory (more reliable for remote players)

```lua
local function getHealthFromAddr(humanoid)
    local addr = tonumber(humanoid.Address)
    if not addr or addr <= 4096 then
        -- fallback to property
        return humanoid.Health, humanoid.MaxHealth
    end
    local okH, health    = pcall(memory_read, "float", addr + HUM.Health)
    local okM, maxHealth = pcall(memory_read, "float", addr + HUM.MaxHealth)
    if okH and okM and health and maxHealth then
        return health, maxHealth
    end
    return humanoid.Health, humanoid.MaxHealth
end
```

---

## GUI Visibility via Memory

```lua
local VISIBLE_OFFSET = 1461

local function isGuiVisible(guiObject)
    local addr = tonumber(guiObject.Address)
    if not addr or addr <= 4096 then return false end
    local ok, byte = pcall(memory_read, "byte", addr + VISIBLE_OFFSET)
    return ok and byte ~= 0
end
```

---

## Sound ID Reading via Memory

```lua
local SOUNDID_OFFSET = 224

local function readSoundId(soundObject)
    local addr = tonumber(soundObject.Address)
    if not addr or addr <= 4096 then return nil end
    local strPtr = readPtr(addr + SOUNDID_OFFSET)
    if not strPtr then return nil end
    local s = readStr(strPtr)
    if not s then return nil end
    return s:match("%d+")  -- extract numeric ID
end
```

---

## UILib (External Library Pattern)

Some scripts load a full UI library from a remote URL instead of using the built-in `UI` global. This is a separate system — the library is downloaded, cached to disk, and loaded via `require()`.

```lua
local LibPath = "C:/matcha/workspace/library.lua"
if not isfile(LibPath) then
    local src = game:HttpGet("https://raw.githubusercontent.com/catowice/p/refs/heads/main/library.lua")
    if src and type(src) == "string" and #src > 100 then
        writefile(LibPath, src)
    end
end
local UILib = require(LibPath)
```

### UILib API (as observed)
```lua
UILib:SetWatermarkEnabled(false)
UILib:SetMenuTitle("Title")
UILib:SetMenuSize(Vector2.new(w, h))
UILib:CenterMenu()
UILib:Step()                            -- must be called every frame in main loop
UILib:Notification("message", seconds)
UILib:CreateSettingsTab("Settings")
UILib:RegisterActivity(function() return "status string" end)

local tab = UILib:Tab("Tab Name")
local sec = tab:Section("Section Name")

sec:Toggle("Label", default, callback)
sec:Slider("Label", default, step, min, max, "unit", callback)
sec:Dropdown("id", {default}, {options}, multiselect, callback)

-- Colorpicker chained on a toggle
local tog = sec:Toggle("Label", default, callback)
tog:AddColorpicker("Label", Color3, showAlpha, callback)

UILib.GetValue("id")
UILib.SetValue("id", val)
```

### Mouse position override
UILib may need its mouse position patched manually:
```lua
local ok, mouse = pcall(function() return Player:GetMouse() end)
UILib._GetMousePos = function(self)
    if ok and mouse then return Vector2.new(mouse.X, mouse.Y) end
    return Vector2.new(0, 0)
end
```

### Accent/theme color
```lua
UILib._theming.accent = Color3.fromRGB(255, 105, 180)
```

### Modifying the Settings tab internals
The Settings tab exposes `_tree` and `_items` — you can remove unwanted menu items:
```lua
local menuItems = UILib._tree["Settings"]._items["Menu"]._items
menuItems[1].label = "Menu key"
table.remove(menuItems, 4)
table.remove(menuItems, 3)
table.remove(menuItems, 2)
```

### Main loop with UILib
```lua
while true do
    UILib:Step()
    -- your per-frame logic
    task.wait()
end
```

---

## MemoryManager Module (External)

Some scripts load a `MemoryManager` module separately from a remote URL. It wraps common memory operations with a cleaner API.

```lua
local MemPath = "C:/matcha/workspace/Modules/MemoryManager.lua"
if not isfile(MemPath) then
    local src = game:HttpGet("https://raw.githubusercontent.com/thelucas128/Macha/refs/heads/main/MemoryManagerFixed.luau")
    if src and type(src) == "string" and #src > 100 then
        writefile(MemPath, src)
    end
end
local MemoryManager = require(MemPath)
```

### Confirmed MemoryManager methods
```lua
MemoryManager.GetRotationMatrix(part)       -- returns rotation matrix table indexed [0]-[8]
MemoryManager.GetGuiObjectRotation(address) -- returns rotation angle of a GuiObject (degrees)
```

`GetRotationMatrix` returns a flat table with indices 0–8 (column-major 3x3):
- `[0],[3],[6]` = right vector components (X axis)
- `[1],[4],[7]` = up vector components (Y axis)
- `[2],[5],[8]` = look/back vector components (Z axis)

Always nil-check the result — returns nil if the part address is invalid.

---

## 3D Box ESP Pattern

Drawing a 3D bounding box requires projecting 8 corners and connecting them with 12 lines.

```lua
local BoxEdges = {
    {1,2},{3,4},{1,3},{2,4},   -- bottom face
    {5,6},{7,8},{5,7},{6,8},   -- top face
    {1,5},{2,6},{3,7},{4,8}    -- vertical edges
}

local function GetCorners3D(part, pos)
    if not pos then return {} end
    local sx, sy, sz = part.Size.X/2, part.Size.Y/2, part.Size.Z/2
    local m = MemoryManager.GetRotationMatrix(part)
    -- fall back to axis-aligned if matrix unavailable
    local r = m and Vector3.new(m[0],m[3],m[6])*sx or Vector3.new(sx,0,0)
    local u = m and Vector3.new(m[1],m[4],m[7])*sy or Vector3.new(0,sy,0)
    local b = m and Vector3.new(m[2],m[5],m[8])*sz or Vector3.new(0,0,sz)
    return {
        pos-r+u+b, pos+r+u+b, pos-r-u+b, pos+r-u+b,
        pos-r+u-b, pos+r+u-b, pos-r-u-b, pos+r-u-b
    }
end

-- Render: project all 8, only draw if all on screen
local pts, allOn = {}, true
for c = 1, 8 do
    local sc, on = WorldToScreen(corners[c])
    if not on or not sc then allOn = false
    else pts[c] = Vector2.new(math.floor(sc.X+0.5), math.floor(sc.Y+0.5)) end
end
if allOn then
    for l = 1, 12 do
        local e = BoxEdges[l]
        lines[l].From, lines[l].To = pts[e[1]], pts[e[2]]
        lines[l].Visible = true
    end
end
```

Cache corners and only recompute when position changes:
```lua
if pos ~= cache.CachedPos then
    cache.Corners = GetCorners3D(part, pos)
    cache.CachedPos = pos
end
```

---

## Circle ESP Pattern (3D Ground Ring)

Drawing a circle on the XZ plane around a character using pre-cached trig multipliers.

```lua
local CircleSegments = 16
local CircleMults = {}
for i = 1, CircleSegments do
    local angle = (i / CircleSegments) * (math.pi * 2)
    CircleMults[i] = { x = math.cos(angle), z = math.sin(angle) }
end

-- Per frame: project each point, connect to next
local radius = 2.5
local pts, allOn = {}, true
for j = 1, CircleSegments do
    local m = CircleMults[j]
    local sc, on = WorldToScreen(pos + Vector3.new(m.x * radius, -3, m.z * radius))
    if not on or not sc then allOn = false
    else pts[j] = Vector2.new(math.floor(sc.X+0.5), math.floor(sc.Y+0.5)) end
end
if allOn then
    for j = 1, CircleSegments do
        local nextIdx = (j % CircleSegments) + 1
        lines[j].From = pts[j]
        lines[j].To   = pts[nextIdx]
        lines[j].Visible = true
    end
end
```

The `-3` Y offset drops the ring to foot level. More segments = smoother circle but more Drawing objects.

---

## Killer Look Tracer (Memory-Based LookVector)

Reading the killer's look direction directly from memory (since CFrame.new is nil) to draw a gradient directional tracer line.

Primitive pointer offset from HumanoidRootPart: `0x148`
Rotation matrix floats within primitive (column-major, same layout as CFrame memory writes):

```lua
-- r02, r12, r22 = third column = look/back vector
local primPtr = memory_read("uintptr_t", hrp.Address + 0x148)
if type(primPtr) == "number" and primPtr > 0x100000 then
    local r02 = memory_read("float", primPtr + 0xC8)
    local r12 = memory_read("float", primPtr + 0xD4)
    local r22 = memory_read("float", primPtr + 0xE0)
    local lookVec = Vector3.new(-r02, -r12, -r22)  -- negate = forward
end
```

Offset summary:
| Component | Offset |
|-----------|--------|
| r02 (look X) | `primPtr + 0xC8` |
| r12 (look Y) | `primPtr + 0xD4` |
| r22 (look Z) | `primPtr + 0xE0` |

Gradient tracer rendering (lerp between two colors across N segments):
```lua
local c1, c2 = Config.TracerColor, Config.TracerColor2
local dr, dg, db = c2.R-c1.R, c2.G-c1.G, c2.B-c1.B
for j = 1, segments do
    local t = j / segments
    lines[j].Color = Color3.new(c1.R + dr*t, c1.G + dg*t, c1.B + db*t)
end
```

---

## Auto Skill Check Pattern

Skill checks are detected by reading `GuiObject` rotation via memory. Comparing the rotating needle angle to the goal zone angle determines when to fire spacebar.

```lua
-- GUI structure expected:
-- PlayerGui > SkillCheckPromptGui > Check > Line (needle)
--                                         > Goal (zone)

local function normalizeAngle(angle)
    angle = angle % 360
    return angle < 0 and angle + 360 or angle
end

local function Autogen()
    local CheckPrompt = PlayerGui:FindFirstChild("SkillCheckPromptGui")
    if not CheckPrompt then hasClicked = false; return end

    local lineObj = CheckPrompt:FindFirstChild("Check") and CheckPrompt.Check:FindFirstChild("Line")
    local goalObj = CheckPrompt:FindFirstChild("Check") and CheckPrompt.Check:FindFirstChild("Goal")
    if not lineObj or not goalObj then return end

    local Rotation     = normalizeAngle(MemoryManager.GetGuiObjectRotation(lineObj.Address))
    local GoalRotation = normalizeAngle(MemoryManager.GetGuiObjectRotation(goalObj.Address))

    -- Perfect zone is ~10 degrees wide, offset 104-114 from goal
    local lower = normalizeAngle(104 + GoalRotation)
    local upper = normalizeAngle(114 + GoalRotation)

    local isPerfect
    if lower < upper then
        isPerfect = Rotation >= lower and Rotation <= upper
    else
        isPerfect = Rotation >= lower or Rotation <= upper  -- wraps 360
    end

    if isPerfect and not hasClicked then
        hasClicked = true
        task.spawn(function()
            task.wait(delay)   -- configurable reaction delay
            keypress(32)       -- spacebar VK
            task.wait(0.02)
            keyrelease(32)
        end)
    else
        hasClicked = false
    end
end
```

Key points: `hasClicked` guard prevents double-firing. The 0.02s gap between press/release is the minimum reliable window. A delay of 0 fires instantly. The perfect zone offset (104–114°) is game-specific and may need adjustment per game.

---

## Drawing Cache / Cleanup Pattern

Per-player Drawing objects should be pooled, not recreated each frame. Clean up when players leave.

```lua
local PlayerDrawings = {}

-- Create on first sight
if not PlayerDrawings[name] then
    PlayerDrawings[name] = {
        Name = Drawing.new("Text"),
        Lines = {}
    }
end

-- Hide when not needed (don't Remove — reuse next frame)
cache.Name.Visible = false

-- Remove when player leaves
for cachedName, cache in pairs(PlayerDrawings) do
    if not activePlayers[cachedName] then
        cache.Name:Remove()
        for _, line in pairs(cache.Lines) do line:Remove() end
        PlayerDrawings[cachedName] = nil
    end
end
```

Also flush drawings for players whose Character/HRP is gone even if they haven't left:
```lua
elseif PlayerDrawings[name] then
    -- Character gone but player still in game — clean up
    PlayerDrawings[name].Name:Remove()
    PlayerDrawings[name] = nil
end
```

---

## Performance: Localize Globals

In tight per-frame loops, localizing frequently-called globals gives a meaningful speedup:

```lua
local math_floor  = math.floor
local math_cos    = math.cos
local math_sin    = math.sin
local math_sqrt   = math.sqrt
local math_pi2    = math.pi * 2
local Vector2_new = Vector2.new
local Vector3_new = Vector3.new
local Color3_new  = Color3.new
local Drawing_new = Drawing.new
local os_clock    = os.clock
local task_spawn  = task.spawn
local task_wait   = task.wait
local WTS         = WorldToScreen
local mem_read    = memory_read
```

Also round Vector2 screen positions to integers to eliminate sub-pixel blur on Drawing lines/text:
```lua
local function roundVec2(v)
    if not v then return Vector2.new(0, 0) end
    return Vector2.new(math.floor(v.X + 0.5), math.floor(v.Y + 0.5))
end
```

---

## Player Info Dump (Server Snapshot)

Dumps all current players' names, display names, UserIDs, and account ages to a file. Filename is keyed to PlaceId so dumps don't overwrite across games.

```lua
local Players = game:GetService("Players")
local fileName = "dump_" .. game.PlaceId .. ".txt"

local function getInfo()
    local lines = {"--- SERVER DUMP: " .. os.date("%X") .. " ---"}
    for _, p in pairs(Players:GetPlayers()) do
        pcall(function()
            table.insert(lines, string.format(
                "User: %s | Display: %s | ID: %d | Age: %s",
                p.Name or "N/A",
                p.DisplayName or "N/A",
                p.UserId or 0,
                p.AccountAge and (tostring(p.AccountAge) .. " days") or "N/A"
            ))
        end)
    end
    table.insert(lines, "-------------------------------------------\n")
    return table.concat(lines, "\n")
end

local function save()
    writefile(fileName, getInfo())
end

-- Auto-save every 30s + on join/leave
task.spawn(function()
    while task.wait(30) do save() end
end)
Players.PlayerAdded:Connect(function() task.wait(2); save() end)
Players.PlayerRemoving:Connect(save)
save()
```

`task.wait(2)` on `PlayerAdded` gives the server time to populate `p.AccountAge` — reading it immediately on join can return nil.

---

## Position HUD (Drawing Text Overlay)

Minimal live coordinate display using Drawing. Note: Drawing Text has a known Matcha bug and may not render — use `notify()` for one-shots, but for a persistent HUD this is the only option.

```lua
local text = Drawing.new("Text")
text.Size     = 16
text.Position = Vector2.new(100, 100)
text.Outline  = true
text.Visible  = true

while true do
    local char = game:GetService("Players").LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local pos = char.HumanoidRootPart.Position
        text.Text = string.format("X: %.1f  Y: %.1f  Z: %.1f", pos.X, pos.Y, pos.Z)
    end
    task.wait()
end
```

`%.1f` in `string.format` rounds to 1 decimal place. Works fine — `string.format` is fully supported.

---

## Model Part Scanner

Prints every Model in workspace that has a Humanoid, along with all its BasePart names. Useful for understanding a new game's character/NPC structure.

```lua
for _, model in pairs(workspace:GetDescendants()) do
    if model:IsA("Model") then
        local humanoid = model:FindFirstChildOfClass("Humanoid")
        if humanoid then
            local parts = {}
            for _, part in pairs(model:GetDescendants()) do
                if part:IsA("BasePart") then
                    table.insert(parts, part.Name)
                end
            end
            if #parts > 0 then
                print(model.Name .. ": " .. table.concat(parts, ", "))
            end
        end
    end
end
```

Pair this with the Workspace Dump pattern for a full picture of a new game.

---

## Game-Specific Recon: Reading SurfaceGui TextLabels

Many games store puzzle/code data in `SurfaceGui > TextLabel` on Parts. Reading them is straightforward:

```lua
local label = part:FindFirstChild("SurfaceGui")
    and part.SurfaceGui:FindFirstChild("TextLabel")
if label then
    local text = label.Text
end
```

### Example: RFID/Blueprint matching pattern
```lua
local correctRfid = workspace.prop_cardReader.main.serial.SurfaceGui.TextLabel.Text

for _, index in ipairs({"1", "2", "3"}) do
    local row = blueprint:FindFirstChild(index)
    if not row then continue end

    local serialLabel = row:FindFirstChild("serial")
        and row.serial:FindFirstChild("SurfaceGui")
        and row.serial.SurfaceGui:FindFirstChild("TextLabel")
    local serialCode = serialLabel and serialLabel.Text

    -- Read multi-part color code
    local colorCode = ""
    local colorsModel = row:FindFirstChild("colors")
    if colorsModel then
        for i = 1, 4 do
            local part = colorsModel:FindFirstChild(tostring(i))
            local lbl = part and part:FindFirstChild("SurfaceGui")
                and part.SurfaceGui:FindFirstChild("TextLabel")
            if lbl then colorCode = colorCode .. lbl.Text end
        end
    end

    if serialCode == correctRfid then
        print("MATCH: " .. serialCode .. " -> " .. colorCode)
    end
end
```

---

## Instant Interact (ProximityPrompt Hold Duration via Memory)

Sets `ProximityPromptHoldDuration` to 0 on all prompts by writing to memory. Offset is loaded from the external offsets JSON.

```lua
local ok, res = pcall(game.HttpGet, game, "https://offsets.ntgetwritewatch.workers.dev/offsets.json")
if ok then
    local data = game:GetService("HttpService"):JSONDecode(res)
    local offset = data and data.ProximityPromptHoldDuraction  -- note: typo in JSON key
        and tonumber(data.ProximityPromptHoldDuraction, 16)

    if offset then
        task.spawn(function()
            while true do
                for _, prompt in ipairs(game:GetDescendants()) do
                    if prompt.ClassName == "ProximityPrompt" then
                        pcall(function()
                            memory_write("float", prompt.Address + offset, 0)
                        end)
                    end
                end
                task.wait(0.1)
            end
        end)
    end
end
```

**Note:** The JSON key is `ProximityPromptHoldDuraction` (typo — missing an 'i'). Match this exactly when reading.
`tonumber(str, 16)` converts the hex string offset to a number.
`game:GetDescendants()` scans the whole game tree — consider scoping to specific services if performance is a concern.

---

## Obfuscation Patterns Encountered in Matcha Scripts

### PTAC VM (Bytecode Interpreter)
Scripts tagged `-- @PTACUNIT` or starting with `VMCall("LOL!..."` are running inside a custom Lua VM. The outer shell deserializes a hex-encoded bytecode string and executes it through a hand-rolled interpreter. Key identifiers:

```lua
-- Entry point pattern
return VMCall("LOL!EB3Q...", GetFEnv(), ...)

-- The VM decodes its bytecode like this:
ByteString = Subg(Sub(ByteString, 5), "..", function(byte)
    -- hex pairs decoded to chars, with run-length compression
end)
```

What this means practically for Matcha scripts:
- The actual logic is invisible in the source. Use the `loadstring` hook pattern (see above) to dump what it loads.
- `getfenv()` / `_ENV` is passed in as `vmenv` — the VM runs in the same environment as the outer script, so it has access to all Matcha globals.
- `unpack` / `table.unpack` compatibility shim is included — the VM handles both Lua 5.1 and 5.2+ style.
- `math.ldexp` is used for float deserialization — confirmed present in Matcha.

### Byte-Array String Obfuscation
A simpler pattern used to hide URLs and strings from static analysis:

```lua
local _0xD = function(b)
    local r = ""
    for i = 1, #b do r = r .. string.char(b[i]) end
    return r
end

-- Usage: encodes a URL as a table of ASCII values
local url = _0xD({104,116,116,112,115,58,47,47,...})
-- Decodes to: "https://..."
```

To decode manually: pass the number array through `string.char` or use `setclipboard` to dump the result at runtime.

### Cache-Busting Pattern
Remote scripts frequently append `?cache=` + `os.time()` to force a fresh fetch, bypassing any CDN caching:

```lua
local url = "https://raw.githubusercontent.com/.../script.lua"
loadstring(game:HttpGet(url .. "?cache=" .. tostring(os.time())))()
```

### GameId / PlaceId Dispatch
Multi-game loaders use `game.GameId` and `game.PlaceId` to branch to different scripts:

```lua
local GID = math.floor(game.GameId)
local PID = math.floor(game.PlaceId)

if PID == math.floor(12345678) then
    -- load script for that specific place
elseif GID == math.floor(98765432) then
    -- load script for that game universe
else
    notify("This game is not supported.", "Error", 5)
end
```

`math.floor` is used on the IDs as an obfuscation/comparison normalisation trick — IDs are already integers but it prevents direct string-matching on the number literals.

---

## Drawing-Based Animated UI (Selector / Splash Screen)

A pattern for building a fully animated, interactive Drawing overlay — no UILib or Matcha UI system required. Used for version selectors, splash screens, or confirmation dialogs.

### Rounded Rectangle via Drawing Primitives
Matcha's Drawing API has no native rounded rectangle. Simulate one with two overlapping Squares and four Circles at the corners:

```lua
-- Rounded rect: two squares + 4 corner circles
local function layoutRoundedRect(s1, s2, c1, c2, c3, c4, x, y, w, h, r)
    r = math.min(r, math.floor(math.min(w, h) / 2))
    s1.Position = Vector2.new(x+r, y);   s1.Size = Vector2.new(w-r*2, h)
    s2.Position = Vector2.new(x, y+r);   s2.Size = Vector2.new(w, h-r*2)
    c1.Position = Vector2.new(x+r,   y+r);   c1.Radius = r
    c2.Position = Vector2.new(x+w-r, y+r);   c2.Radius = r
    c3.Position = Vector2.new(x+r,   y+h-r); c3.Radius = r
    c4.Position = Vector2.new(x+w-r, y+h-r); c4.Radius = r
end
```

Border lines connect the flat edges between corners:
```lua
bL1.From = Vector2.new(x+r, y);    bL1.To = Vector2.new(x+w-r, y)     -- top
bL2.From = Vector2.new(x+r, y+h);  bL2.To = Vector2.new(x+w-r, y+h)   -- bottom
bL3.From = Vector2.new(x, y+r);    bL3.To = Vector2.new(x, y+h-r)     -- left
bL4.From = Vector2.new(x+w, y+r);  bL4.To = Vector2.new(x+w, y+h-r)  -- right
```

### Scale-In Animation
Animate the card growing from 0 to full size on open:
```lua
local dur = 0.4
local t0 = tick()
while tick()-t0 < dur do
    local p = (tick()-t0)/dur
    local ep = 1-(1-p)^3    -- cubic ease-out
    local cw = math.max(2, math.floor(CW*ep))
    local ch = math.max(2, math.floor(CH*ep))
    layoutRoundedRect(...)
    task.wait(0.016)         -- ~60fps
end
```

### Slide-In + Fade-In with Staggered Delays
Content items animate in with a vertical slide + opacity fade, each with its own delay:
```lua
-- contentItems = list of {obj, bx, by, delay}
local slideOff = 20   -- pixels above final position to start from

while tick()-t0 < totalDur do
    local elapsed = tick() - t0
    for _, ci in ipairs(contentItems) do
        local t = elapsed - (ci.delay or 0)
        if t < 0 then
            ci.obj.Transparency = 0
        elseif t < slideDur then
            local ep = 1-(1-(t/slideDur))^3
            ci.obj.Transparency = ep
            ci.obj.Position = Vector2.new(ci.bx, ci.by + math.floor(slideOff*(1-ep)))
        else
            ci.obj.Transparency = 1
            ci.obj.Position = Vector2.new(ci.bx, ci.by)
        end
    end
    task.wait(0.016)
end
```

### Button Hover / Scale Interpolation
Buttons track a `curScale` value and lerp toward `targetScale` each frame:
```lua
btn.targetScale = onHover and 1.08 or 1.0
btn.curScale = btn.curScale + (btn.targetScale - btn.curScale) * 0.15
layoutBtn(btn, btn.curScale)
```

Label color lerps between a default color and accent on hover:
```lua
local function lerpColor(c1, c2, t)
    return Color3.fromRGB(
        math.floor(c1.R*255 + (c2.R*255-c1.R*255)*t),
        math.floor(c1.G*255 + (c2.G*255-c1.G*255)*t),
        math.floor(c1.B*255 + (c2.B*255-c1.B*255)*t)
    )
end
btn.lbl.Color = lerpColor(btn.defCol, accentCol, easeOut(btn.lblT))
```

### Click Bounce Animation
On click, the selected button plays a multi-phase squish bounce:
```lua
-- t goes 0→1 over 0.4s
if     t < 0.25 then sc = lerp(1.08, 0.88, easeOut(t/0.25))       -- squish in
elseif t < 0.55 then sc = lerp(0.88, 1.06, easeOut((t-0.25)/0.3)) -- overshoot
elseif t < 0.80 then sc = lerp(1.06, 0.97, easeOut((t-0.55)/0.25))-- settle
else                 sc = lerp(0.97, 1.00, easeOut((t-0.80)/0.2))  -- rest
end
```

### Draggable Card
The whole card can be dragged — track mouse delta each frame:
```lua
-- State
local _drg, _dox, _doy = false, 0, 0

-- On press + not on button: start drag
if m1 and onCard and not onButton then
    _drg = true; _dox = mx; _doy = my
end

-- Each frame while dragging: translate everything
if _drg then
    local dx, dy = mx - _dox, my - _doy
    scx = scx + dx; scy = scy + dy
    -- re-layout all Drawing objects with new offsets
    _dox = mx; _doy = my
end
```

### Mouse Hit Testing
Matcha has no UI hit testing. Do it manually with a bounds check:
```lua
local function ins(mx, my, rx, ry, rw, rh)
    return mx >= rx and mx <= rx+rw and my >= ry and my <= ry+rh
end
```

### Drawing Object Pool
All Drawing objects are stored in a single list for easy bulk cleanup:
```lua
local dObjs = {}
local function mk(typ, props)
    local o = Drawing.new(typ)
    for k, v in pairs(props) do o[k] = v end
    table.insert(dObjs, o)
    return o
end
local function cleanup()
    for _, o in ipairs(dObjs) do pcall(function() o:Remove() end) end
    table.clear(dObjs)
end
```

### Fade-Out on Exit
Before loading the selected script, fade out all Drawing objects:
```lua
local snap = {}
for _, o in ipairs(dObjs) do table.insert(snap, o) end
local t0, dur = tick(), 0.3
while tick()-t0 < dur do
    local ep = (tick()-t0)/dur
    for _, o in ipairs(snap) do pcall(function() o.Transparency = 1-ep end) end
    task.wait(0.016)
end
cleanup()
```

### Font Fallback Pattern
Prefer SystemBold but fall back gracefully if it's unavailable:
```lua
local FNT = Drawing.Fonts.Monospace
pcall(function() FNT = Drawing.Fonts.System end)
local FNTB = FNT
pcall(function() FNTB = Drawing.Fonts.SystemBold end)
```

### Launching the Selected Script
After the UI closes, fetch and run the chosen URL:
```lua
task.spawn(loadstring(game:HttpGet(url .. "?cache=" .. tostring(os.time()))))
while true do task.wait(60) end   -- keep script alive
```

`task.spawn` wraps the `loadstring` call so errors in the loaded script don't crash this loader. The infinite `task.wait` loop at the end prevents the loader from exiting (which could terminate spawned threads depending on Matcha behavior).

### ZIndex Layering Convention (observed)
| Layer | ZIndex | Content |
|-------|--------|---------|
| Card background | 50 | Filled squares/circles |
| Card border | 51 | Border lines |
| Content | 52 | Text, buttons |
| Button labels | 54 | On top of button fills |

---

## General Tips

- Wrap ALL property reads on remote instances in pcall — Matcha can throw instead of returning nil
- `setclipboard()` is useful for extracting data out of the game during debugging
- `notify()` is the only reliable on-screen text (Drawing Text is bugged)
- `task.wait()` stacking causes timing issues — keep waits simple
- `String.format` works normally
- `os.clock()` works for timing
- `math` library works fully
- No auto-timeout needed in loops since there's no RunService timeout
- `continue` keyword works inside loops
- Scripts crash with "attempt to index nil with 'Name'" when LocalPlayer is nil at script start — always defer LocalPlayer reads into functions
- `pcall` returning true doesn't guarantee the value isn't nil in Matcha — always check both: `if ok and value then`
- `Color3.fromRGB` takes 0-255, `Color3.new` takes 0-1, `BrickColor.Color` returns 0-1 — don't mix them up
- `IsA()` can return unexpected values — always compare with `== true` explicitly
