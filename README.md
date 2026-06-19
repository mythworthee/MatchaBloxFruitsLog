# MatchaBloxFruitsLog

--[[
══════════════════════════════════════════════════════════════════════════════
                          BLOX FRUITS NPC COMPANION
                              ( Matcha Edition )
══════════════════════════════════════════════════════════════════════════════

  Target executor : Matcha (external LuaVM)
  Game            : Blox Fruits (Roblox)
  Use case        : SINGLEPLAYER TESTING in a private/empty server
  UI Library      : library.lua ( UILib, made for Matcha )

  ───────────────────────────────────────────────────────────────────────────
  HOW THE LEVEL FARM WORKS (the 3-step flow you asked for)
  ───────────────────────────────────────────────────────────────────────────

  Step 1 — ROUTE TO ISLAND IF QUEST GIVER MISSING
      • readLocalLevel() -> look up the right QuestRoute for your level
      • findQuestGiver() walks workspace.NPCs / workspace for the giver model
      • if giver is missing -> tween to the island's "ApproachCFrame" so the
        island streams in and the giver NPC loads -> re-scan

  Step 2 — ACCEPT QUEST ( GUI clicks, since Matcha can't fire RemoteEvents )
      • tween to within 6 studs of the giver
      • click the giver NPC (mousemoveabs + mouse1click, both 3-arg per Matcha)
      • wait for "Please select a quest." dialog to open in PlayerGui.Main.Dialogue
      • scan dialog buttons for one whose Text contains the target enemy name
        (e.g. "Gorilla") -> click it
      • wait 1.70 seconds (your requested delay) for the Confirm button to appear
      • scan for a TextButton whose Text == "Confirm" / "Accept" / "Start Quest"
      • click it -> quest accepted -> Main.Quest.Visible becomes true

  Step 3 — KILL MOBS
      • tween to 30 studs above the route's EnemySpawnCFrame
      • scan workspace.Enemies for the target mob name
      • when found, tween to 6 studs above the mob's Head
      • autoclicker + hitbox extender do the actual damage (separate toggles)
      • loop until the quest counter in Main.Quest is fulfilled
      • when quest disappears -> re-route to next quest giver

  ───────────────────────────────────────────────────────────────────────────
  LOAD ORDER
  ───────────────────────────────────────────────────────────────────────────
  1. library.lua       ( defines global `UILib` )
  2. blox_fruits_companion.lua  ( this file — uses `UILib` )

══════════════════════════════════════════════════════════════════════════════
]]

-- ============================================================================
-- 0. SETUP — load UILib if it isn't already
-- ============================================================================

local LibPath = "C:/matcha/workspace/library.lua"
if not isfile(LibPath) then
    -- fall back to whatever folder this script was run from
    LibPath = "library.lua"
end

local UILib = UILib  -- may already be set by loader
if type(UILib) ~= "table" and isfile(LibPath) then
    local ok, lib = pcall(function() return require(LibPath) end)
    if ok and type(lib) == "table" then
        UILib = lib
    end
end

if type(UILib) ~= "table" then
    if type(notify) == "function" then
        notify("Could not load UILib. Place library.lua in C:/matcha/workspace/ first.", "Blox Companion", 8)
    end
    return
end

_G.UILib = UILib

-- ============================================================================
-- 1. CLEANUP PREVIOUS INSTANCE
-- ============================================================================

if _G.__BloxFruitsCompanion_Cleanup then
    pcall(_G.__BloxFruitsCompanion_Cleanup)
end

-- ============================================================================
-- 2. SERVICES & LOCALS
-- ============================================================================

local Players         = game:GetService("Players")
local ReplicatedStore = game:GetService("ReplicatedStorage")
local Workspace       = game:GetService("Workspace")

-- localize hot globals (per Matcha perf docs)
local task_wait       = task.wait
local task_spawn      = task.spawn
local os_clock        = os.clock
local math_floor      = math.floor
local math_sqrt       = math.sqrt
local math_pi         = math.pi
local math_random     = math.random
local math_cos        = math.cos
local math_sin        = math.sin
local math_pow        = math.pow
local string_lower    = string.lower
local string_find     = string.find
local string_format   = string.format
local string_gsub     = string.gsub
local string_match    = string.match
local string_sub      = string.sub
local table_concat    = table.concat
local table_insert    = table.insert
local Vector3_new     = Vector3.new
local Vector2_new     = Vector2.new
local CFrame_new      = CFrame.new
local CFrame_lookAt   = CFrame.lookAt
local CFrame_Angles   = CFrame.Angles
local Drawing_new     = Drawing.new
local WorldToScreen   = WorldToScreen

-- ============================================================================
-- 2.5. FORWARD DECLARATIONS
-- ============================================================================
-- Matcha has a 200 local register limit. To avoid hitting it, we use GLOBALS
-- for functions that are only called at runtime (not at file load time).
-- Globals have no register limit in Matcha.

-- Input helpers (used by guiClickAt which is used early — keep as locals)
local safeMouseMoveAbs
local safeMouse1Click
local safeMouse1HoldClick
local isValidScreenCoord
local guiClickAt

-- GUI scanner helpers (converted to globals to save registers)
-- getViewportSize, numberField, getGuiCenter, getQuestGuiRoots, etc.
-- are assigned as globals via `functionName = function() ... end`

-- ============================================================================
-- 3. CONFIG
-- ============================================================================

local Config = {
    Title            = "Blox Fruits",
    Accent           = Color3.fromRGB(255, 170, 43),
    TextColor        = Color3.fromRGB(255, 240, 220),
    Watermark        = true,

    -- NPC ESP / scan
    RefreshRate      = 0.10,
    Range            = 600,
    MaxShown         = 80,
    YOffset          = 3,
    NpcFilter        = "",
    NpcNametags      = true,
    NpcHealthbars    = true,
    NpcBoxESP        = true,
    DynamicHealthColor = true,
    StaticHealthColor  = Color3.fromRGB(80, 255, 120),
    ShowDistance     = true,
    ShowHealthText   = false,
    HealthbarWidth   = 42,
    HealthbarHeight  = 5,

    -- Hitbox extender
    HeadHitbox       = true,
    HeadSize         = 30,

    -- Level Farm
    LevelProcess     = false,
    LevelMode        = "Quest",          -- "Quest" or "Manual"
    LevelAutoRoute   = true,
    LevelTarget      = "",               -- manual override for "Manual" mode
    LevelMoveToTarget= true,
    LevelAutoClick   = false,           -- autoclicker OFF by default (toggle in UI)
    SmartRescan      = true,            -- rescan NPCs when kill count increases
    LevelClickHold   = 0.03,
    LevelClickInterval = 0.30,            -- click once every 0.3 seconds
    LevelDistance    = 0,                -- hover directly on mob head
    LevelHeightOffset= 20,               -- 20 studs above mob
    LevelTweenSpeed  = 600,
    LevelTweenEasing = "linear",
    LevelDelay       = 0.08,             -- per-step delay

    -- Quest giver
    QuestGiverEnabled   = true,
    QuestGiverOverride  = "",             -- manual giver name override
    QuestGiverDistance  = 6,
    QuestGiverHeightOffset = 0,
    QuestGuiDetect      = true,
    QuestCoordinateFallback = true,
    QuestUiDelay        = 0.02,
    QuestOptionClickTime= 6.0,            -- how long to wait for option button
    QuestConfirmDelay   = 0.50,           -- delay between option click and confirm scan
    QuestConfirmScanTime= 3.0,            -- how long to scan for Confirm button
    QuestRefreshTime    = 45,

    -- Enemy spawn approach
    EnemyApproachHeight = 30,             -- float 30 studs above spawn (your spec)
    EnemyApproachSpeed  = 600,

    -- Teleport waypoints
    TeleportTweenSpeed  = 600,
    TeleportHeightOffset= 6,

    -- Chests
    ChestTween       = false,
    ChestFilter      = "",  -- empty = show ALL chest types (silver, gold, diamond)
    ChestRange       = 5000,
    ChestTweenSpeed  = 600,
    ChestHeightOffset= 3,
    ChestCooldown    = 8,
    ChestVisualRange = false,
    ChestESP         = false,

    -- Fruit ESP
    FruitNametags    = false,
    FruitTagColor    = Color3.fromRGB(255, 140, 220),
    FruitRange       = 1500,
    FruitFilter      = "Fruit",
    AutoBuso         = false,
    ServerHop        = false,
    ServerHopDelay   = 30,
    FruitShowDistance= true,
    FruitShowRarity  = true,

    -- Internal
    LastAction       = "Loaded",
    Folders          = { "Enemies", "NPCs" },
}

-- ============================================================================
-- 4. STATE
-- ============================================================================

local Running      = true
local Cleaned      = false
local StartedAt    = os_clock()
local NearbyNPCs   = {}
local Labels       = {}
local Bars         = {}
local Boxes         = {}  -- Box ESP: 4 lines per NPC key
local OriginalHeads= {}

-- Module-level Color3 constants — avoids creating a new Color3 object on
-- every NPC every frame in renderNpcOverlays. Color3.fromRGB is cheap but
-- not free, and the render loop processes 10-50 NPCs at 60fps.
local COLOR_BLACK        = Color3.fromRGB(0, 0, 0)
local COLOR_WHITE        = Color3.fromRGB(255, 255, 255)
local COLOR_HEALTHBAR_BG = Color3.fromRGB(20, 20, 20)
local COLOR_BOSS_RED     = Color3.fromRGB(255, 50, 50)
local COLOR_DEFAULT_HP   = Color3.fromRGB(80, 255, 120)

-- Quest Progress HUD (multi-line)
local QuestHUD = Drawing.new("Text")
QuestHUD.Visible = false
QuestHUD.Center = false
QuestHUD.Outline = true
QuestHUD.Size = 16
QuestHUD.Color = Color3.fromRGB(255, 235, 130)
QuestHUD.ZIndex = 90
QuestHUD.Position = Vector2.new(20, 80)
pcall(function() QuestHUD.Font = Drawing.Fonts.System end)

local QuestHUD2 = Drawing.new("Text")
QuestHUD2.Visible = false
QuestHUD2.Center = false
QuestHUD2.Outline = true
QuestHUD2.Size = 14
QuestHUD2.Color = Color3.fromRGB(180, 220, 255)
QuestHUD2.ZIndex = 90
QuestHUD2.Position = Vector2.new(20, 100)
pcall(function() QuestHUD2.Font = Drawing.Fonts.System end)

local QuestHUD3 = Drawing.new("Text")
QuestHUD3.Visible = false
QuestHUD3.Center = false
QuestHUD3.Outline = true
QuestHUD3.Size = 14
QuestHUD3.Color = Color3.fromRGB(180, 255, 180)
QuestHUD3.ZIndex = 90
QuestHUD3.Position = Vector2.new(20, 118)
pcall(function() QuestHUD3.Font = Drawing.Fonts.System end)

-- Range visualization circle (for chest range)
local RANGE_SEGMENTS = 40
local rangeCircleLines = {}
for i = 1, RANGE_SEGMENTS do
    local l = Drawing.new("Line")
    l.Thickness = 1
    l.Color = Color3.fromRGB(100, 200, 255)
    l.Visible = false
    l.ZIndex = 70
    table.insert(rangeCircleLines, l)
end

-- Chest ESP labels
local chestLabels = {}

-- Box ESP helper: get or create 4 lines for a key
function getBoxLines(key)
    if not Boxes[key] then
        local lines = {}
        for i = 1, 4 do
            local l = Drawing.new("Line")
            l.Visible = false
            l.Thickness = 1
            l.Color = Color3.fromRGB(255, 255, 255)
            l.ZIndex = 78
            table.insert(lines, l)
        end
        Boxes[key] = lines
    end
    return Boxes[key]
end

function hideAllBoxes()
    for _, lines in pairs(Boxes) do
        for _, l in ipairs(lines) do pcall(function() l.Visible = false end) end
    end
end

function removeAllBoxes()
    for _, lines in pairs(Boxes) do
        for _, l in ipairs(lines) do pcall(function() l:Remove() end) end
    end
    Boxes = {}
end

local LevelState = {
    Target             = "none",
    CurrentLevel       = 0,
    LevelSource        = "none",
    RouteIsland        = "none",
    RouteQuestGiver    = "none",
    RouteEnemy         = "none",
    RouteMinLevel      = 0,
    RouteOption        = 1,
    RouteQuestName     = "none",      -- e.g. "JungleQuest"
    RouteLevelIdx      = 1,           -- 1 or 2
    RouteQuestGiverCFrame = nil,      -- Vector3
    RouteEnemySpawnCFrame = nil,      -- Vector3
    RouteKillCount     = 0,
    QuestAccepted      = false,
    QuestPhase         = "route",     -- route|to_island|to_giver|quest_ui|fight|giver_missing|no_root|disabled
    QuestGiverObject   = "none",
    QuestGiverPosition = nil,
    LastQuestAt        = 0,
    LastStepAt         = 0,
    StepCount          = 0,
    ClickHeld          = false,
    ClickReleaseAt     = 0,
    LastClickAt        = 0,
    TweenSignal        = nil,
    TweenPurpose       = "none",      -- none|quest_giver|enemy|teleport|island
    EnemyApproachDone  = false,
    QuestLog           = {},
    -- failure-tracking fields (prevents infinite accept loop)
    LastAcceptFailAt   = 0,         -- os_clock() of last failed acceptQuestUi
    AcceptFailCount    = 0,         -- consecutive failures
    AcceptCooldownSecs = 6,         -- min seconds between accept attempts
    AcceptMaxFails     = 3,         -- auto-pause after this many consecutive fails
    AutoPaused         = false,     -- true when LevelProcess auto-paused due to fails
}

local NearbyChests = {}
local ChestVisited = {}
local ChestState = {
    Target       = "none",
    TargetKey    = "none",
    Count        = 0,
    LastScanAt   = 0,
    LastVisitAt  = 0,
    TweenSignal  = nil,
}

-- ============================================================================
-- GUI PATH CACHE
--   Quest tracker reads happen on every frame (HUD update + levelProcessStep).
--   Each uncached read does 5-7 chained safeFind() calls, each wrapped in a
--   pcall (expensive). We cache the PlayerGui → Main → Quest → Container →
--   QuestTitle → Title chain and refresh it periodically. If any cached
--   reference becomes invalid (parent set to nil, object destroyed), the
--   cache is invalidated automatically on next read.
-- ============================================================================

local GuiCache = {
    PlayerGui     = nil,
    Main          = nil,
    Quest         = nil,
    Container     = nil,
    QuestTitle    = nil,
    Title         = nil,
    LastRefreshAt = 0,
}

-- Refresh the GUI cache. Cheap to call — only walks the chain once.
refreshGuiCache = function()
    local lp = getLocalPlayer()
    if not lp then
        GuiCache.PlayerGui  = nil
        GuiCache.Main       = nil
        GuiCache.Quest      = nil
        GuiCache.Container  = nil
        GuiCache.QuestTitle = nil
        GuiCache.Title      = nil
        return
    end
    local pg = safeFind(lp, "PlayerGui")
    GuiCache.PlayerGui = pg
    if not pg then return end
    local main = safeFind(pg, "Main")
    GuiCache.Main = main
    if not main then return end
    local quest = safeFind(main, "Quest")
    GuiCache.Quest = quest
    if not quest then return end
    local container = safeFind(quest, "Container")
    GuiCache.Container = container
    if not container then return end
    local qt = safeFind(container, "QuestTitle")
    GuiCache.QuestTitle = qt
    if not qt then return end
    GuiCache.Title = safeFind(qt, "Title")
end

-- Returns the cached Title TextLabel (or nil if cache is empty/invalid).
getCachedQuestTitle = function()
    return GuiCache.Title
end

-- Returns the cached Quest frame (or nil).
getCachedQuestFrame = function()
    return GuiCache.Quest
end

-- Invalidate the cache (forces a full refresh on next access).
invalidateGuiCache = function()
    GuiCache.PlayerGui  = nil
    GuiCache.Main       = nil
    GuiCache.Quest      = nil
    GuiCache.Container  = nil
    GuiCache.QuestTitle = nil
    GuiCache.Title      = nil
end

-- ============================================================================
-- 5. QUEST ROUTES TABLE (Sea 1 + 2 + 3, 84 bands total)
--    Sources: Baokhanh208/Infinite, xZPUHigh/swswsw, pokelok/krahb, ServerSad
--    Each route has: Min level, Island name, QuestGiver name, Enemy name,
--    QuestName string (for CommF_ remote if you ever switch to internal),
--    LevelIdx (1 or 2), QuestGiverCFrame (Vector3), EnemySpawnCFrame (Vector3),
--    KillCount (default per quest — actual count is read from quest tracker)
-- ============================================================================

local QuestRoutes = {
    -- ── SEA 1 (Lv 1–700) ──────────────────────────────────────────────────
    {Min=1,    Island="Starter Island",    QuestGiver="Bandit Quest Giver",     Enemy="Bandit",                 QuestName="BanditQuest1",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(1059.37, 15.45, 1550.42),  EnemySpawnPos=Vector3.new(1045.96, 27.00, 1560.82)},
    {Min=10,   Island="Jungle",            QuestGiver="Adventurer",             Enemy="Monkey",                 QuestName="JungleQuest",   LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-1598.09, 35.55, 153.38),  EnemySpawnPos=Vector3.new(-1448.52, 67.85, 11.47)},
    {Min=15,   Island="Jungle",            QuestGiver="Adventurer",             Enemy="Gorilla",                QuestName="JungleQuest",   LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-1598.09, 35.55, 153.38),  EnemySpawnPos=Vector3.new(-1129.88, 40.46, -525.42)},
    {Min=30,   Island="Pirate Village",    QuestGiver="Pirate Adventurer",      Enemy="Pirate",                 QuestName="BuggyQuest1",   LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-1141.07, 4.10, 3831.55),  EnemySpawnPos=Vector3.new(-1103.51, 13.75, 3896.09)},
    {Min=40,   Island="Pirate Village",    QuestGiver="Pirate Adventurer",      Enemy="Brute",                  QuestName="BuggyQuest1",   LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-1141.07, 4.10, 3831.55),  EnemySpawnPos=Vector3.new(-1140.08, 14.81, 4322.92)},
    {Min=60,   Island="Desert",            QuestGiver="Desert Adventurer",      Enemy="Desert Bandit",          QuestName="DesertQuest",   LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(894.49, 5.14, 4392.43),    EnemySpawnPos=Vector3.new(924.80, 6.45, 4481.59)},
    {Min=75,   Island="Desert",            QuestGiver="Desert Adventurer",      Enemy="Desert Officer",         QuestName="DesertQuest",   LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(894.49, 5.14, 4392.43),    EnemySpawnPos=Vector3.new(1608.28, 8.61, 4371.01)},
    {Min=90,   Island="Frozen Village",    QuestGiver="Villager",               Enemy="Snow Bandit",            QuestName="SnowQuest",     LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(1389.74, 88.15, -1298.91), EnemySpawnPos=Vector3.new(1354.35, 87.27, -1393.95)},
    {Min=100,  Island="Frozen Village",    QuestGiver="Villager",               Enemy="Snowman",                QuestName="SnowQuest",     LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(1389.74, 88.15, -1298.91), EnemySpawnPos=Vector3.new(1201.64, 144.58, -1550.07)},
    {Min=120,  Island="Marine Fortress",   QuestGiver="Marine",                 Enemy="Chief Petty Officer",    QuestName="MarineQuest2",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-5039.59, 27.35, 4324.68), EnemySpawnPos=Vector3.new(-4881.23, 22.65, 4273.75)},
    {Min=150,  Island="Skylands",          QuestGiver="Sky Adventurer",         Enemy="Sky Bandit",             QuestName="SkyQuest",      LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-4839.53, 716.37, -2619.44),EnemySpawnPos=Vector3.new(-4953.21, 295.74, -2899.23)},
    {Min=175,  Island="Skylands",          QuestGiver="Sky Adventurer",         Enemy="Dark Master",            QuestName="SkyQuest",      LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-4839.53, 716.37, -2619.44),EnemySpawnPos=Vector3.new(-5259.84, 391.40, -2229.04)},
    {Min=190,  Island="Prison",            QuestGiver="Jail Keeper",            Enemy="Prisoner",               QuestName="PrisonerQuest", LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(5308.93, 1.66, 475.12),    EnemySpawnPos=Vector3.new(5098.97, -0.32, 474.24)},
    {Min=210,  Island="Prison",            QuestGiver="Jail Keeper",            Enemy="Dangerous Prisoner",     QuestName="PrisonerQuest", LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(5308.93, 1.66, 475.12),    EnemySpawnPos=Vector3.new(5654.56, 15.63, 866.30)},
    {Min=250,  Island="Colosseum",         QuestGiver="Colosseum Quest Giver",  Enemy="Toga Warrior",           QuestName="ColosseumQuest",LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-1580.05, 6.35, -2986.48), EnemySpawnPos=Vector3.new(-1820.21, 51.68, -2740.67)},
    {Min=275,  Island="Colosseum",         QuestGiver="Colosseum Quest Giver",  Enemy="Gladiator",              QuestName="ColosseumQuest",LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-1580.05, 6.35, -2986.48), EnemySpawnPos=Vector3.new(-1292.84, 56.38, -3339.03)},
    {Min=300,  Island="Magma Village",     QuestGiver="The Mayor",              Enemy="Military Soldier",       QuestName="MagmaQuest",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-5313.37, 10.95, 8515.29), EnemySpawnPos=Vector3.new(-5411.16, 11.08, 8454.29)},
    {Min=325,  Island="Magma Village",     QuestGiver="The Mayor",              Enemy="Military Spy",           QuestName="MagmaQuest",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-5313.37, 10.95, 8515.29), EnemySpawnPos=Vector3.new(-5802.87, 86.26, 8828.86)},
    {Min=375,  Island="Underwater City",   QuestGiver="King Neptune",           Enemy="Fishman Warrior",        QuestName="FishmanQuest",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(61122.65, 18.50, 1569.40), EnemySpawnPos=Vector3.new(60878.30, 18.48, 1543.76)},
    {Min=400,  Island="Underwater City",   QuestGiver="King Neptune",           Enemy="Fishman Commando",       QuestName="FishmanQuest",  LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(61122.65, 18.50, 1569.40), EnemySpawnPos=Vector3.new(61922.63, 18.48, 1493.93)},
    {Min=450,  Island="Upper Skylands",    QuestGiver="Mole",                   Enemy="God's Guard",            QuestName="SkyExp1Quest",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-4721.89, 843.87, -1949.97),EnemySpawnPos=Vector3.new(-4710.04, 845.28, -1927.31)},
    {Min=475,  Island="Upper Skylands",    QuestGiver="Mole",                   Enemy="Shanda",                 QuestName="SkyExp1Quest",  LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-7859.10, 5544.19, -381.48),EnemySpawnPos=Vector3.new(-7678.49, 5566.40, -497.22)},
    {Min=525,  Island="Upper Skylands",    QuestGiver="Sky Quest Giver 2",      Enemy="Royal Squad",            QuestName="SkyExp2Quest",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-7906.82, 5634.66, -1411.99),EnemySpawnPos=Vector3.new(-7624.25, 5658.13, -1467.35)},
    {Min=550,  Island="Upper Skylands",    QuestGiver="Sky Quest Giver 2",      Enemy="Royal Soldier",          QuestName="SkyExp2Quest",  LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-7906.82, 5634.66, -1411.99),EnemySpawnPos=Vector3.new(-7836.75, 5645.66, -1790.62)},
    {Min=625,  Island="Fountain City",     QuestGiver="Freezeburg Quest Giver", Enemy="Galley Pirate",          QuestName="FountainQuest", LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(5259.82, 37.35, 4050.03),  EnemySpawnPos=Vector3.new(5551.02, 78.90, 3930.41)},
    {Min=650,  Island="Fountain City",     QuestGiver="Freezeburg Quest Giver", Enemy="Galley Captain",         QuestName="FountainQuest", LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(5259.82, 37.35, 4050.03),  EnemySpawnPos=Vector3.new(5441.95, 42.50, 4950.09)},

    -- ── SEA 2 (Lv 700–1500) ───────────────────────────────────────────────
    {Min=700,  Island="Kingdom of Rose",   QuestGiver="Area 1 Quest Giver",     Enemy="Raider",                QuestName="Area1Quest",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-429.54, 71.77, 1836.18),  EnemySpawnPos=Vector3.new(-728.33, 52.78, 2345.77)},
    {Min=725,  Island="Kingdom of Rose",   QuestGiver="Area 1 Quest Giver",     Enemy="Mercenary",             QuestName="Area1Quest",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-429.54, 71.77, 1836.18),  EnemySpawnPos=Vector3.new(-1004.32, 80.16, 1424.62)},
    {Min=775,  Island="Kingdom of Rose",   QuestGiver="Area 2 Quest Giver",     Enemy="Swan Pirate",            QuestName="Area2Quest",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(638.44, 71.77, 918.28),    EnemySpawnPos=Vector3.new(1068.66, 137.61, 1322.11)},
    {Min=800,  Island="Kingdom of Rose",   QuestGiver="Area 2 Quest Giver",     Enemy="Factory Staff",          QuestName="Area2Quest",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(632.70, 73.11, 918.67),    EnemySpawnPos=Vector3.new(73.08, 81.86, -27.47)},
    {Min=875,  Island="Green Zone",        QuestGiver="Marine Quest Giver",     Enemy="Marine Lieutenant",      QuestName="MarineQuest3",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-2440.80, 71.71, -3216.07),EnemySpawnPos=Vector3.new(-2821.37, 75.90, -3070.09)},
    {Min=900,  Island="Green Zone",        QuestGiver="Marine Quest Giver",     Enemy="Marine Captain",         QuestName="MarineQuest3",  LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-2440.80, 71.71, -3216.07),EnemySpawnPos=Vector3.new(-1861.23, 80.18, -3254.70)},
    {Min=950,  Island="Graveyard Island",  QuestGiver="Graveyard Quest Giver",  Enemy="Zombie",                 QuestName="ZombieQuest",   LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-5497.06, 47.59, -795.24), EnemySpawnPos=Vector3.new(-5657.78, 78.97, -928.69)},
    {Min=975,  Island="Graveyard Island",  QuestGiver="Graveyard Quest Giver",  Enemy="Vampire",                QuestName="ZombieQuest",   LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-5497.06, 47.59, -795.24), EnemySpawnPos=Vector3.new(-6037.67, 32.18, -1340.66)},
    {Min=1000, Island="Snow Mountain",     QuestGiver="Snow Quest Giver",       Enemy="Snow Trooper",           QuestName="SnowMountainQuest",LevelIdx=1,KillCount=8, QuestGiverPos=Vector3.new(609.86, 400.12, -5372.26), EnemySpawnPos=Vector3.new(549.15, 427.39, -5563.70)},
    {Min=1050, Island="Snow Mountain",     QuestGiver="Snow Quest Giver",       Enemy="Winter Warrior",         QuestName="SnowMountainQuest",LevelIdx=2,KillCount=10,QuestGiverPos=Vector3.new(609.86, 400.12, -5372.26), EnemySpawnPos=Vector3.new(1142.75, 475.64, -5199.42)},
    {Min=1100, Island="Hot and Cold",      QuestGiver="Ice Quest Giver",        Enemy="Lab Subordinate",        QuestName="IceSideQuest",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-6064.07, 15.24, -4902.98),EnemySpawnPos=Vector3.new(-5707.47, 15.95, -4513.39)},
    {Min=1125, Island="Hot and Cold",      QuestGiver="Ice Quest Giver",        Enemy="Horned Warrior",         QuestName="IceSideQuest",  LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-6064.07, 15.24, -4902.98),EnemySpawnPos=Vector3.new(-6341.37, 15.95, -5723.16)},
    {Min=1175, Island="Hot and Cold",      QuestGiver="Fire Quest Giver",       Enemy="Magma Ninja",            QuestName="FireSideQuest", LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-5428.03, 15.06, -5299.43),EnemySpawnPos=Vector3.new(-5449.67, 76.66, -5808.20)},
    {Min=1200, Island="Hot and Cold",      QuestGiver="Fire Quest Giver",       Enemy="Lava Pirate",            QuestName="FireSideQuest", LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-5428.03, 15.06, -5299.43),EnemySpawnPos=Vector3.new(-5213.33, 49.74, -4701.45)},
    {Min=1250, Island="Cursed Ship",       QuestGiver="Rear Crew Quest Giver",  Enemy="Ship Deckhand",          QuestName="ShipQuest1",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(1037.80, 125.09, 32911.60),EnemySpawnPos=Vector3.new(1212.01, 150.79, 33059.25)},
    {Min=1275, Island="Cursed Ship",       QuestGiver="Rear Crew Quest Giver",  Enemy="Ship Engineer",          QuestName="ShipQuest1",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(1037.80, 125.09, 32911.60),EnemySpawnPos=Vector3.new(919.48, 43.54, 32779.97)},
    {Min=1300, Island="Cursed Ship",       QuestGiver="Front Crew Quest Giver", Enemy="Ship Steward",           QuestName="ShipQuest2",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(968.81, 125.09, 33244.13), EnemySpawnPos=Vector3.new(919.44, 129.56, 33436.04)},
    {Min=1325, Island="Cursed Ship",       QuestGiver="Front Crew Quest Giver", Enemy="Ship Officer",           QuestName="ShipQuest2",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(968.81, 125.09, 33244.13), EnemySpawnPos=Vector3.new(1036.02, 181.44, 33315.73)},
    {Min=1350, Island="Ice Castle",        QuestGiver="Frost Quest Giver",      Enemy="Arctic Warrior",         QuestName="FrostQuest",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(5667.66, 26.80, -6486.09), EnemySpawnPos=Vector3.new(5966.25, 62.97, -6179.38)},
    {Min=1375, Island="Ice Castle",        QuestGiver="Frost Quest Giver",      Enemy="Snow Lurker",            QuestName="FrostQuest",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(5667.66, 26.80, -6486.09), EnemySpawnPos=Vector3.new(5407.07, 69.19, -6880.88)},
    {Min=1425, Island="Forgotten Island",  QuestGiver="Forgotten Quest Giver",  Enemy="Sea Soldier",            QuestName="ForgottenQuest",LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-3054.44, 235.54, -10142.82),EnemySpawnPos=Vector3.new(-3028.22, 64.67, -9775.43)},
    {Min=1450, Island="Forgotten Island",  QuestGiver="Forgotten Quest Giver",  Enemy="Water Fighter",          QuestName="ForgottenQuest",LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-3054.44, 235.54, -10142.82),EnemySpawnPos=Vector3.new(-3352.90, 285.02, -10534.84)},

    -- ── SEA 3 (Lv 1500–2525+) ─────────────────────────────────────────────
    {Min=1500, Island="Port Town",         QuestGiver="Pirate Port Quest Giver",Enemy="Pirate Millionaire",    QuestName="PiratePortQuest",LevelIdx=1,KillCount=8,  QuestGiverPos=Vector3.new(-290.07, 42.90, 5581.59),  EnemySpawnPos=Vector3.new(-245.99, 47.31, 5584.10)},
    {Min=1525, Island="Port Town",         QuestGiver="Pirate Port Quest Giver",Enemy="Pistol Billionaire",    QuestName="PiratePortQuest",LevelIdx=2,KillCount=10, QuestGiverPos=Vector3.new(-290.07, 42.90, 5581.59),  EnemySpawnPos=Vector3.new(-187.33, 86.24, 6013.51)},
    {Min=1575, Island="Hydra Island",      QuestGiver="Dragon Crew Quest Giver",Enemy="Dragon Crew Warrior",    QuestName="AmazonQuest",   LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(5832.84, 51.68, -1101.52), EnemySpawnPos=Vector3.new(6141.14, 51.35, -1340.74)},
    {Min=1600, Island="Hydra Island",      QuestGiver="Dragon Crew Quest Giver",Enemy="Dragon Crew Archer",     QuestName="AmazonQuest",   LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(5833.11, 51.60, -1103.07), EnemySpawnPos=Vector3.new(6616.42, 441.77, 446.05)},
    {Min=1625, Island="Hydra Island",      QuestGiver="Hydra Town Quest Giver", Enemy="Hydra Enforcer",         QuestName="AmazonQuest2",  LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(5446.88, 601.63, 749.46),  EnemySpawnPos=Vector3.new(4685.26, 735.81, 815.34)},
    {Min=1650, Island="Hydra Island",      QuestGiver="Hydra Town Quest Giver", Enemy="Venomous Assailant",     QuestName="AmazonQuest2",  LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(5446.88, 601.63, 749.46),  EnemySpawnPos=Vector3.new(4729.09, 590.44, -36.98)},
    {Min=1700, Island="Great Tree",        QuestGiver="Marine Tree Quest Giver",Enemy="Marine Commodore",      QuestName="MarineTreeIsland",LevelIdx=1,KillCount=8, QuestGiverPos=Vector3.new(2180.54, 27.82, -6741.55), EnemySpawnPos=Vector3.new(2286.01, 73.13, -7159.81)},
    {Min=1725, Island="Great Tree",        QuestGiver="Marine Tree Quest Giver",Enemy="Marine Rear Admiral",   QuestName="MarineTreeIsland",LevelIdx=2,KillCount=10,QuestGiverPos=Vector3.new(2179.99, 28.73, -6740.06), EnemySpawnPos=Vector3.new(3656.77, 160.52, -7001.60)},
    {Min=1775, Island="Floating Turtle",   QuestGiver="Turtle Adventure Quest Giver",Enemy="Fishman Raider",    QuestName="DeepForestIsland3",LevelIdx=1,KillCount=8,QuestGiverPos=Vector3.new(-10581.66, 330.87, -8761.19),EnemySpawnPos=Vector3.new(-10407.53, 331.76, -8368.52)},
    {Min=1800, Island="Floating Turtle",   QuestGiver="Turtle Adventure Quest Giver",Enemy="Fishman Captain",   QuestName="DeepForestIsland3",LevelIdx=2,KillCount=10,QuestGiverPos=Vector3.new(-10581.66, 330.87, -8761.19),EnemySpawnPos=Vector3.new(-10994.70, 352.38, -9002.11)},
    {Min=1825, Island="Floating Turtle",   QuestGiver="Deep Forest Quest Giver",Enemy="Forest Pirate",         QuestName="DeepForestIsland",LevelIdx=1,KillCount=8, QuestGiverPos=Vector3.new(-13234.04, 331.49, -7625.40),EnemySpawnPos=Vector3.new(-13274.48, 332.38, -7769.58)},
    {Min=1850, Island="Floating Turtle",   QuestGiver="Deep Forest Quest Giver",Enemy="Mythological Pirate",   QuestName="DeepForestIsland",LevelIdx=2,KillCount=10,QuestGiverPos=Vector3.new(-13234.04, 331.49, -7625.40),EnemySpawnPos=Vector3.new(-13680.61, 501.08, -6991.19)},
    {Min=1900, Island="Floating Turtle",   QuestGiver="Forest Area 2 Quest Giver",Enemy="Jungle Pirate",       QuestName="DeepForestIsland2",LevelIdx=1,KillCount=8,QuestGiverPos=Vector3.new(-12680.38, 389.97, -9902.02),EnemySpawnPos=Vector3.new(-12256.16, 331.74, -10485.84)},
    {Min=1925, Island="Floating Turtle",   QuestGiver="Forest Area 2 Quest Giver",Enemy="Musketeer Pirate",    QuestName="DeepForestIsland2",LevelIdx=2,KillCount=10,QuestGiverPos=Vector3.new(-12680.38, 389.97, -9902.02),EnemySpawnPos=Vector3.new(-13457.90, 391.55, -9859.18)},
    {Min=1975, Island="Haunted Castle",    QuestGiver="Haunted Castle Quest Giver",Enemy="Reborn Skeleton",    QuestName="HauntedQuest1", LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-9479.22, 141.22, 5566.09), EnemySpawnPos=Vector3.new(-8763.72, 165.72, 6159.86)},
    {Min=2000, Island="Haunted Castle",    QuestGiver="Haunted Castle Quest Giver",Enemy="Living Zombie",      QuestName="HauntedQuest1", LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-9479.22, 141.22, 5566.09), EnemySpawnPos=Vector3.new(-10144.13, 138.63, 5838.09)},
    {Min=2025, Island="Haunted Castle",    QuestGiver="Castle Quest Giver 2",   Enemy="Demonic Soul",           QuestName="HauntedQuest2", LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-9516.99, 172.02, 6078.47), EnemySpawnPos=Vector3.new(-9505.87, 172.10, 6158.99)},
    {Min=2050, Island="Haunted Castle",    QuestGiver="Castle Quest Giver 2",   Enemy="Possessed Mummy",        QuestName="HauntedQuest2", LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-9516.99, 172.02, 6078.47), EnemySpawnPos=Vector3.new(-9582.02, 6.25, 6205.48)},
    {Min=2075, Island="Sea of Treats",     QuestGiver="Peanut Quest Giver",     Enemy="Peanut Scout",           QuestName="NutsIslandQuest",LevelIdx=1,KillCount=8,  QuestGiverPos=Vector3.new(-2104.39, 38.10, -10194.22),EnemySpawnPos=Vector3.new(-2143.24, 47.72, -10029.99)},
    {Min=2100, Island="Sea of Treats",     QuestGiver="Peanut Quest Giver",     Enemy="Peanut President",       QuestName="NutsIslandQuest",LevelIdx=2,KillCount=10, QuestGiverPos=Vector3.new(-2104.39, 38.10, -10194.22),EnemySpawnPos=Vector3.new(-1859.35, 38.10, -10422.43)},
    {Min=2125, Island="Sea of Treats",     QuestGiver="Ice Cream Quest Giver",  Enemy="Ice Cream Chef",         QuestName="IceCreamIslandQuest",LevelIdx=1,KillCount=8,QuestGiverPos=Vector3.new(-820.65, 65.82, -10965.80),EnemySpawnPos=Vector3.new(-872.25, 65.82, -10919.96)},
    {Min=2150, Island="Sea of Treats",     QuestGiver="Ice Cream Quest Giver",  Enemy="Ice Cream Commander",    QuestName="IceCreamIslandQuest",LevelIdx=2,KillCount=10,QuestGiverPos=Vector3.new(-820.65, 65.82, -10965.80),EnemySpawnPos=Vector3.new(-558.06, 112.05, -11290.77)},
    {Min=2200, Island="Sea of Treats",     QuestGiver="Cake Quest Giver 1",     Enemy="Cookie Crafter",         QuestName="CakeQuest1",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-2021.32, 37.80, -12028.73),EnemySpawnPos=Vector3.new(-2374.14, 37.80, -12125.31)},
    {Min=2225, Island="Sea of Treats",     QuestGiver="Cake Quest Giver 1",     Enemy="Cake Guard",             QuestName="CakeQuest1",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-2021.32, 37.80, -12028.73),EnemySpawnPos=Vector3.new(-1598.31, 43.77, -12244.58)},
    {Min=2250, Island="Sea of Treats",     QuestGiver="Cake Quest Giver 2",     Enemy="Baking Staff",           QuestName="CakeQuest2",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-1927.92, 37.80, -12842.54),EnemySpawnPos=Vector3.new(-1887.81, 77.62, -12998.35)},
    {Min=2275, Island="Sea of Treats",     QuestGiver="Cake Quest Giver 2",     Enemy="Head Baker",             QuestName="CakeQuest2",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-1927.92, 37.80, -12842.54),EnemySpawnPos=Vector3.new(-2216.19, 82.88, -12869.29)},
    {Min=2300, Island="Sea of Treats",     QuestGiver="Chocolate Quest Giver 1",Enemy="Cocoa Warrior",         QuestName="ChocQuest1",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(233.23, 29.88, -12201.23), EnemySpawnPos=Vector3.new(-21.55, 80.57, -12352.39)},
    {Min=2325, Island="Sea of Treats",     QuestGiver="Chocolate Quest Giver 1",Enemy="Chocolate Bar Battler", QuestName="ChocQuest1",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(233.23, 29.88, -12201.23), EnemySpawnPos=Vector3.new(582.59, 77.19, -12463.16)},
    {Min=2350, Island="Sea of Treats",     QuestGiver="Chocolate Quest Giver 2",Enemy="Sweet Thief",           QuestName="ChocQuest2",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(150.51, 30.69, -12774.50), EnemySpawnPos=Vector3.new(165.19, 76.06, -12600.84)},
    {Min=2375, Island="Sea of Treats",     QuestGiver="Chocolate Quest Giver 2",Enemy="Candy Rebel",           QuestName="ChocQuest2",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(150.51, 30.69, -12774.50), EnemySpawnPos=Vector3.new(134.87, 77.25, -12876.55)},
    {Min=2400, Island="Sea of Treats",     QuestGiver="Candy Cane Quest Giver", Enemy="Candy Pirate",           QuestName="CandyQuest1",   LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-1150.04, 20.38, -14446.33),EnemySpawnPos=Vector3.new(-1310.50, 26.02, -14562.40)},
    {Min=2425, Island="Sea of Treats",     QuestGiver="Candy Cane Quest Giver", Enemy="Snow Demon",             QuestName="CandyQuest1",   LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-1150.04, 20.38, -14446.33),EnemySpawnPos=Vector3.new(-880.20, 71.25, -14538.61)},
    {Min=2450, Island="Tiki Outpost",      QuestGiver="Tiki Quest Giver 1",     Enemy="Isle Outlaw",            QuestName="TikiQuest1",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-16545.94, 55.69, -173.23),EnemySpawnPos=Vector3.new(-16120.60, 116.52, -103.04)},
    {Min=2475, Island="Tiki Outpost",      QuestGiver="Tiki Quest Giver 1",     Enemy="Island Boy",             QuestName="TikiQuest1",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-16545.94, 55.69, -173.23),EnemySpawnPos=Vector3.new(-16751.31, 121.23, -264.02)},
    {Min=2500, Island="Tiki Outpost",      QuestGiver="Tiki Quest Giver 2",     Enemy="Sun-kissed Warrior",     QuestName="TikiQuest2",    LevelIdx=1, KillCount=8,  QuestGiverPos=Vector3.new(-16539.08, 55.69, 1051.57),EnemySpawnPos=Vector3.new(-16294.67, 32.79, 1062.49)},
    {Min=2525, Island="Tiki Outpost",      QuestGiver="Tiki Quest Giver 2",     Enemy="Isle Champion",          QuestName="TikiQuest2",    LevelIdx=2, KillCount=10, QuestGiverPos=Vector3.new(-16539.08, 55.69, 1051.57),EnemySpawnPos=Vector3.new(-16933.21, 93.35, 999.45)},
}

-- ============================================================================
-- 5.5. BOSS BLACKLIST
--     These enemy names are BOSSES — the script will NEVER target them.
--     When a quest giver offers a boss as an option (usually Option3),
--     the script skips it and clicks the highest-level non-boss option.
--     Source: Blox Fruits wiki boss list + autofarm script data.
-- ============================================================================

local BOSS_ENEMY_NAMES = {
    -- Sea 1 bosses
    "Gorilla King", "Bobby", "Yeti", "Mob Leader", "Vice Admiral",
    "Warden", "Chief Warden", "Swan", "Magma Admiral", "Fishman Lord",
    "Wysper", "Thunder God", "Cyborg",
    "Saber Expert", "The Saw", "Greybeard",
    -- Sea 2 bosses
    "Diamond", "Jeremy", "Fajita", "Don Swan", "Smoke Admiral",
    "Cursed Captain", "Darkbeard", "Order", "Awakened Ice Admiral",
    "Tide Keeper",
    -- Sea 3 bosses
    "Stone", "Island Empress", "Kilo Admiral", "Captain Elephant",
    "Beautiful Pirate", "Cake Queen", "Longma", "Soul Reaper",
    "rip_indra",
    -- Generic boss name patterns (lowercase check)
    -- We also check for "Boss" or "Raid Boss" in the display name
}

-- Check if an enemy name is a boss.
-- Returns true if the name matches a known boss OR contains "Boss"/"Raid Boss".
isBossEnemy = function(name)
    if type(name) ~= "string" then return false end
    local lower = string_lower(name)
    -- Check explicit boss list
    for _, bossName in ipairs(BOSS_ENEMY_NAMES) do
        if lower == string_lower(bossName) then
            return true
        end
        -- Also check substring (e.g. "Gorilla King [Lv. 25]" contains "Gorilla King")
        if string_find(lower, string_lower(bossName), 1, true) then
            return true
        end
    end
    -- Check for "Boss" or "Raid Boss" in the name
    if string_find(lower, "boss", 1, true) then return true end
    if string_find(lower, "raid boss", 1, true) then return true end
    return false
end

-- ============================================================================
-- 5.55. BOAT DEALER BLACKLIST
--      Per user request: if a candidate quest giver's name contains
--      "Boat Dealer" (case-insensitive), reject it immediately. Boat Dealers
--      are shop NPCs that never give quests, but their names often contain
--      keywords that match quest giver names (e.g. "Advanced Marines Boat
--      Dealer" matched the wanted giver "Marine" via loose substring match).
--
--      The word-boundary matching in nameMatchesQuestGiver already prevents
--      "Marine" from matching "Marines" — this is a second line of defense
--      for any future boat dealer variants.
-- ============================================================================

-- Returns true if the candidate NPC name contains "boat dealer".
-- Case-insensitive substring match.
isShopNpc = function(name)
    if type(name) ~= "string" then return false end
    local lower = string_lower(name)
    if string_find(lower, "boat dealer", 1, true) then
        return true
    end
    return false
end

-- ============================================================================
-- 5.6. KNOWN DEVIL FRUITS (Blox Fruits)
--     In Blox Fruits, spawned Devil Fruits appear in the world as Model
--     instances named "Fruit" with a child "Handle" (the visible mesh).
--     The actual fruit type name is stored in a child object OR derivable
--     from a BillboardGui inside the Handle.
--     Source: devforum.roblox.com (Workspace.DevilFruitSpawner.SpawnedDFS),
--             rscripts.net snippet (v.Name=="Fruit" + v:FindFirstChild("Handle")),
--             infocheats.net (Workspace.Fruits listener).
--     This table is used for:
--       1) Matching child object names to identify the fruit type
--       2) Coloring the ESP label by rarity tier
-- ============================================================================

-- Format: [lowercase_name] = rarity_tier
-- Tiers: "mythical" (rare/expensive), "legendary", "rare", "uncommon", "common"
local KNOWN_FRUITS = {
    -- Beast fruits (most coveted)
    ["dragon"]      = "mythical",
    ["leopard"]     = "mythical",
    ["kitsune"]     = "mythical",
    ["t-rex"]       = "mythical",
    ["yeti"]        = "mythical",
    ["dragon fruit"]= "mythical",
    -- Mythical natural/elemental
    ["control"]     = "mythical",
    ["dough"]       = "mythical",
    ["shadow"]      = "mythical",
    ["venom"]       = "mythical",
    ["gravity"]     = "mythical",
    ["sound"]       = "mythical",
    ["phoenix"]     = "mythical",
    ["portal"]      = "mythical",
    ["spirit"]      = "mythical",
    ["blizzard"]    = "mythical",
    ["gas"]         = "mythical",
    ["bull"]        = "mythical",
    ["tractor"]     = "mythical",
    -- Legendary
    ["buddha"]      = "legendary",
    ["rumble"]      = "legendary",
    ["paw"]         = "legendary",
    ["quake"]       = "legendary",
    ["spider"]      = "legendary",
    ["love"]        = "legendary",
    ["magma"]       = "legendary",
    ["dark"]        = "legendary",
    ["light"]       = "legendary",
    ["sand"]         = "legendary",
    ["string"]      = "legendary",
    ["diamond"]     = "legendary",
    -- Rare
    ["flame"]       = "rare",
    ["ice"]         = "rare",
    ["falcon"]      = "rare",
    ["rubber"]      = "rare",
    ["revive"]      = "rare",
    ["smoke"]       = "rare",
    ["spike"]       = "rare",
    -- Uncommon
    ["blade"]       = "uncommon",
    ["spring"]      = "uncommon",
    ["bomb"]        = "uncommon",
    ["kilo"]        = "uncommon",
    -- Common
    ["chop"]        = "common",
    ["spin"]        = "common",
    ["human"]       = "common",
    ["bird: phoenix"] = "mythical", -- alternate naming convention
    ["human: buddha"] = "legendary",
    ["bird: falcon"]  = "rare",
}

-- Returns (displayName, rarityTier) for a fruit object.
-- Tries multiple sources in order:
--   1. Child objects with known fruit names (e.g. child named "Bomb" → "Bomb")
--   2. BillboardGui TextLabel inside the model/Handle showing fruit name
--   3. Strip "Fruit" from the model name (e.g. "BombFruit" → "Bomb")
--   4. Fallback to model name as-is
getFruitDisplayName = function(fruitObj, modelName)
    local name = tostring(modelName or "Fruit")
    local lower

    -- Strategy 1: scan children for any with a known fruit name
    for _, child in ipairs(safeChildren(fruitObj)) do
        local cn = safeProp(child, "Name") or ""
        lower = string_lower(cn)
        if KNOWN_FRUITS[lower] then
            return cn, KNOWN_FRUITS[lower]
        end
    end

    -- Strategy 2: look for a BillboardGui -> TextLabel with the fruit name
    local handle = safeFind(fruitObj, "Handle")
    if handle then
        for _, d in ipairs(safeDescendants(handle)) do
            local dn = safeProp(d, "Name") or ""
            lower = string_lower(dn)
            if KNOWN_FRUITS[lower] then
                return dn, KNOWN_FRUITS[lower]
            end
            -- Also check Text property of TextLabels (Blox Fruits sometimes stores
            -- the visible fruit name as the Text, not the Name)
            local cn = safeProp(d, "ClassName") or ""
            if cn == "TextLabel" then
                local txt
                local ok, t = pcall(function() return d.Text end)
                if ok then txt = t end
                if type(txt) == "string" and #txt > 0 and #txt < 40 then
                    local lt = string_lower(txt)
                    -- strip common prefixes/suffixes like "Fruit", " Devil Fruit"
                    local stripped = txt
                    stripped = string_gsub(stripped, "[Dd]evil ", "")
                    stripped = string_gsub(stripped, "[Ff]ruit", "")
                    stripped = string_gsub(stripped, "^%s+", "")
                    stripped = string_gsub(stripped, "%s+$", "")
                    local ls = string_lower(stripped)
                    if KNOWN_FRUITS[ls] then
                        return stripped, KNOWN_FRUITS[ls]
                    end
                    -- Also try the raw text lowercased (for "Bird: Phoenix" style)
                    if KNOWN_FRUITS[lt] then
                        return txt, KNOWN_FRUITS[lt]
                    end
                end
            end
        end
    end

    -- Strategy 3: strip "Fruit" from the model name
    -- Handles "BombFruit", "Bomb Fruit", "Dragon Fruit" etc.
    local stripped = name
    stripped = string_gsub(stripped, "[Ff]ruit", "")
    stripped = string_gsub(stripped, "^%s+", "")
    stripped = string_gsub(stripped, "%s+$", "")
    if #stripped > 0 then
        lower = string_lower(stripped)
        if KNOWN_FRUITS[lower] then
            return stripped, KNOWN_FRUITS[lower]
        end
        -- Even if not in known list, return the stripped name with unknown rarity
        return stripped, "unknown"
    end

    -- Strategy 4: just return the original name
    lower = string_lower(name)
    if KNOWN_FRUITS[lower] then
        return name, KNOWN_FRUITS[lower]
    end
    return name, "unknown"
end

-- Returns a Color3 for a given fruit rarity tier.
getFruitColor = function(rarity)
    if rarity == "mythical"  then return Color3.fromRGB(255, 80, 80)  end -- red
    if rarity == "legendary" then return Color3.fromRGB(255, 200, 50) end -- gold
    if rarity == "rare"      then return Color3.fromRGB(120, 200, 255) end -- blue
    if rarity == "uncommon"  then return Color3.fromRGB(120, 255, 130) end -- green
    if rarity == "common"    then return Color3.fromRGB(200, 200, 200) end -- gray
    return Color3.fromRGB(255, 140, 220) -- unknown -> pink default
end

-- ============================================================================
-- 6. ISLAND WAYPOINTS (for "float to island to load quest giver" step)
--    These are approach points ~50 studs above each island's center.
-- ============================================================================

local IslandWaypoints = {
    ["Starter Island"]      = Vector3.new(1060, 50, 1547),
    ["Jungle"]              = Vector3.new(-1600, 80, 152),
    ["Pirate Village"]      = Vector3.new(-1122, 50, 3855),
    ["Desert"]              = Vector3.new(1094, 60, 4192),
    ["Frozen Village"]      = Vector3.new(1199, 80, -1211),
    ["Marine Fortress"]     = Vector3.new(-4500, 50, 4260),
    ["Skylands"]            = Vector3.new(-4970, 750, -2620),
    ["Prison"]              = Vector3.new(4857, 50, 740),
    ["Colosseum"]           = Vector3.new(-1420, 50, -3014),
    ["Magma Village"]       = Vector3.new(-5247, 50, 8504),
    ["Underwater City"]     = Vector3.new(61164, 50, 1819),
    ["Upper Skylands"]      = Vector3.new(-7950, 5600, -320),
    ["Fountain City"]       = Vector3.new(5132, 60, 4037),
    ["Kingdom of Rose"]     = Vector3.new(-394, 150, 1245),
    ["Green Zone"]          = Vector3.new(-2448, 100, -3210),
    ["Graveyard Island"]    = Vector3.new(-5500, 80, -795),
    ["Snow Mountain"]       = Vector3.new(1384, 500, -4990),
    ["Hot and Cold"]        = Vector3.new(-6020, 50, -4900),
    ["Cursed Ship"]         = Vector3.new(923, 150, 32852),
    ["Ice Castle"]          = Vector3.new(5400, 60, -6236),
    ["Forgotten Island"]    = Vector3.new(-3050, 270, -10150),
    ["Port Town"]           = Vector3.new(-290, 80, 5577),
    ["Hydra Island"]        = Vector3.new(5229, 650, 345),
    ["Great Tree"]          = Vector3.new(2276, 50, -6493),
    ["Floating Turtle"]     = Vector3.new(-11050, 380, -8760),
    ["Haunted Castle"]      = Vector3.new(-9515, 180, 5538),
    ["Sea of Treats"]       = Vector3.new(-2140, 90, -12380),
    ["Tiki Outpost"]        = Vector3.new(-16500, 80, -172),
}

-- ============================================================================
-- 7. SAFE WRAPPERS (Matcha pcall-everything pattern)
-- ============================================================================

function safeFind(parent, name)
    if not parent then return nil end
    local ok, result = pcall(function() return parent:FindFirstChild(name) end)
    if ok then return result end
    return nil
end

function safeChildren(parent)
    if not parent then return {} end
    local ok, result = pcall(function() return parent:GetChildren() end)
    if ok and type(result) == "table" then return result end
    return {}
end

function safeDescendants(parent)
    if not parent then return {} end
    local ok, result = pcall(function() return parent:GetDescendants() end)
    if ok and type(result) == "table" then return result end
    return {}
end

function safeProp(obj, prop)
    if not obj then return nil end
    local ok, result = pcall(function() return obj[prop] end)
    if ok then return result end
    return nil
end

function safeSet(obj, prop, value)
    if not obj then return false end
    local ok = pcall(function() obj[prop] = value end)
    return ok == true
end

function safeFullName(obj)
    if not obj then return "" end
    local ok, result = pcall(function() return obj:GetFullName() end)
    if ok and type(result) == "string" then return result end
    return safeProp(obj, "Name") or ""
end

function readValueObjectNumber(obj)
    if not obj then return nil end
    local value = safeProp(obj, "Value")
    if type(value) == "number" then return value end
    local asNumber = tonumber(value)
    if asNumber then return asNumber end
    return nil
end

function setStatus(text)
    Config.LastAction = text or "Updated"
end

function notifyUser(text, seconds)
    setStatus(text)
    pcall(function() UILib:Notification(text, seconds or 4) end)
end

function qlog(msg)
    local entry = "[" .. string_format("%.3f", os_clock()) .. "] " .. tostring(msg)
    local log = LevelState.QuestLog
    log[#log + 1] = entry
    -- Trim in bulk when buffer gets 50 entries over the cap. This avoids
    -- the O(n) table.remove(t, 1) shift running on EVERY log call after
    -- overflow. With this approach, the trim runs once every ~50 logs.
    if #log > 250 then
        -- Drop the oldest 50 entries in one operation.
        local kept = {}
        for i = 51, #log do
            kept[#kept + 1] = log[i]
        end
        LevelState.QuestLog = kept
    end
    print("[BFCompanion] " .. entry)
end

-- ============================================================================
-- 8. PLAYER / CHARACTER HELPERS
-- ============================================================================

function getLocalPlayer()
    return Players.LocalPlayer
end

function getLocalRoot()
    local lp = getLocalPlayer()
    if not lp then return nil end
    local char = safeProp(lp, "Character")
    if not char then return nil end
    return safeFind(char, "HumanoidRootPart") or safeFind(char, "Torso") or safeFind(char, "UpperTorso")
end

function readLocalLevel()
    local lp = getLocalPlayer()
    if not lp then return 0, "none" end

    -- Blox Fruits: Players.LocalPlayer.Data.Level.Value
    local data = safeFind(lp, "Data")
    local levelObject = data and safeFind(data, "Level")
    local level = readValueObjectNumber(levelObject)
    if level then return math_floor(level), "Data.Level" end

    -- fallback: leaderstats.Level
    local leaderstats = safeFind(lp, "leaderstats")
    levelObject = leaderstats and safeFind(leaderstats, "Level")
    level = readValueObjectNumber(levelObject)
    if level then return math_floor(level), "leaderstats.Level" end

    return 0, "none"
end

function distSq(a, b)
    local dx = b.X - a.X
    local dy = b.Y - a.Y
    local dz = b.Z - a.Z
    return dx*dx + dy*dy + dz*dz
end

function dist3(a, b)
    return math_sqrt(distSq(a, b))
end

-- ============================================================================
-- 9. QUEST ROUTE LOOKUP
-- ============================================================================

function getQuestRouteForLevel(level)
    local selected = QuestRoutes[1]
    for _, route in ipairs(QuestRoutes) do
        if level >= route.Min then
            -- Skip boss routes — never target bosses for farming
            if not isBossEnemy(route.Enemy) then
                selected = route
            end
        else
            break
        end
    end
    return selected
end

function getQuestOptionForRoute(route)
    -- Count how many routes share the same QuestGiver BEFORE this one
    -- (givers with 2 quests: option 1 = first quest, option 2 = second quest)
    local option = 1
    for _, other in ipairs(QuestRoutes) do
        if other == route then
            return option
        end
        if other.QuestGiver == route.QuestGiver and other.Island == route.Island then
            option = option + 1
        end
    end
    return option
end

function updateQuestRoute()
    local route = getQuestRouteForLevel(LevelState.CurrentLevel)
    if not route then return nil end

    local previousEnemy = LevelState.RouteEnemy

    LevelState.RouteIsland            = route.Island
    LevelState.RouteQuestGiver        = route.QuestGiver
    LevelState.RouteEnemy             = route.Enemy
    LevelState.RouteMinLevel          = route.Min
    LevelState.RouteOption            = getQuestOptionForRoute(route)
    LevelState.RouteQuestName         = route.QuestName
    LevelState.RouteLevelIdx          = route.LevelIdx
    LevelState.RouteQuestGiverCFrame  = route.QuestGiverPos
    LevelState.RouteEnemySpawnCFrame  = route.EnemySpawnPos
    LevelState.RouteKillCount         = route.KillCount or 8

    if previousEnemy ~= route.Enemy then
        LevelState.QuestAccepted     = false
        LevelState.QuestPhase        = "route"
        LevelState.LastQuestAt       = 0
        LevelState.EnemyApproachDone = false
    end

    if Config.LevelMode == "Quest" and Config.LevelAutoRoute then
        Config.LevelTarget = route.Enemy
        Config.NpcFilter   = route.Enemy
    end

    return route
end

-- ============================================================================
-- 10. NPC SCAN + HITBOX EXTENDER
-- ============================================================================

function nameMatches(name)
    if type(Config.NpcFilter) ~= "string" or #Config.NpcFilter < 1 then return true end
    local a = string_lower(tostring(name or ""))
    local b = string_lower(Config.NpcFilter)
    return string_find(a, b, 1, true) ~= nil
end

function makeKey(obj)
    local addr = safeProp(obj, "Address")
    if addr then return tostring(addr) end
    return tostring(obj)
end

function getNPCInfo(model)
    if not model then return nil end
    local hum = safeFind(model, "Humanoid")
    if not hum then return nil end

    local hp = safeProp(hum, "Health")
    local maxHp = safeProp(hum, "MaxHealth")
    if type(hp) == "number" and hp <= 0 then return nil end
    if type(maxHp) ~= "number" or maxHp <= 0 then maxHp = 100 end
    if type(hp) ~= "number" then hp = maxHp end

    local head = safeFind(model, "Head")
    if not head then return nil end
    local pos = safeProp(head, "Position")
    if not pos then return nil end

    local name = safeProp(model, "Name") or "NPC"
    if not nameMatches(name) then return nil end

    return {
        Model     = model,
        Head      = head,
        Humanoid  = hum,    -- cached for live health re-reads in renderNpcOverlays
        Pos       = pos,
        Name      = name,
        Health    = hp,
        MaxHealth= maxHp,
        Key      = makeKey(model),
        HeadKey  = makeKey(head),
    }
end

-- Head hitbox extender
function rememberHead(head, headKey)
    if OriginalHeads[headKey] then
        OriginalHeads[headKey].Head = head
        return true
    end
    local originalSize = safeProp(head, "Size")
    if not originalSize then return false end
    OriginalHeads[headKey] = {
        Head = head,
        Size = originalSize,
        CanCollide   = safeProp(head, "CanCollide"),
        Massless     = safeProp(head, "Massless"),
        Transparency = safeProp(head, "Transparency"),
    }
    return true
end

function restoreHead(headKey)
    local data = OriginalHeads[headKey]
    if not data or not data.Head then
        OriginalHeads[headKey] = nil
        return
    end
    if data.Size         then safeSet(data.Head, "Size",         data.Size) end
    if data.CanCollide   ~= nil then safeSet(data.Head, "CanCollide",   data.CanCollide) end
    if data.Massless     ~= nil then safeSet(data.Head, "Massless",     data.Massless) end
    if data.Transparency ~= nil then safeSet(data.Head, "Transparency", data.Transparency) end
    OriginalHeads[headKey] = nil
end

function restoreAllHeads()
    local keys = {}
    for key, _ in pairs(OriginalHeads) do table.insert(keys, key) end
    for _, key in ipairs(keys) do restoreHead(key) end
end

function applyHeadHitbox(head, headKey)
    if not Config.HeadHitbox then return end
    if not head or not headKey then return end
    if not rememberHead(head, headKey) then return end
    local s = Config.HeadSize
    safeSet(head, "Size", Vector3_new(s, s, s))
    safeSet(head, "CanCollide", false)
    safeSet(head, "Massless", true)
end

function applyHitboxesToNearby()
    if not Config.HeadHitbox then return end
    for _, npc in ipairs(NearbyNPCs) do applyHeadHitbox(npc.Head, npc.HeadKey) end
end

function restoreInactiveHeads(activeHeads)
    local keys = {}
    for key, _ in pairs(OriginalHeads) do
        if not activeHeads[key] then table.insert(keys, key) end
    end
    for _, key in ipairs(keys) do restoreHead(key) end
end

function scanNPCs()
    local root = getLocalRoot()
    local rootPos = root and safeProp(root, "Position")
    local found = {}
    local seenModels = {}
    local activeHeads = {}

    local currentLevel, levelSource = readLocalLevel()
    LevelState.CurrentLevel = currentLevel
    LevelState.LevelSource  = levelSource
    updateQuestRoute()

    if rootPos then
        local rangeSq = Config.Range * Config.Range
        for _, folderName in ipairs(Config.Folders) do
            local folder = safeFind(Workspace, folderName)
            if folder then
                for _, child in ipairs(safeChildren(folder)) do
                    local info = getNPCInfo(child)
                    if info and not seenModels[info.Key] then
                        local d2 = distSq(rootPos, info.Pos)
                        if d2 <= rangeSq then
                            info.Distance = math_sqrt(d2)
                            table.insert(found, info)
                            seenModels[info.Key] = true
                            activeHeads[info.HeadKey] = true
                            if Config.HeadHitbox then
                                applyHeadHitbox(info.Head, info.HeadKey)
                            end
                        end
                    end
                end
            end
        end
    end

    NearbyNPCs = found
    if Config.HeadHitbox then restoreInactiveHeads(activeHeads) end
end

-- ============================================================================
-- 10.5. LIGHT RESCAN — scan ONLY Workspace.Enemies[RouteEnemy]
--   Called when the quest kill count goes up by +1 (a mob died). Instead of
--   re-running the full scanNPCs() (which iterates ALL enemies + NPCs in
--   Config.Range), this only iterates Workspace.Enemies children whose name
--   matches the current route enemy. Much cheaper, and ensures we acquire
--   the next target immediately after a kill.
--
--   MERGE behavior: non-route-enemy NPCs already in NearbyNPCs are kept.
--   Only the route-enemy entries are replaced with the fresh scan. This
--   preserves ESP for other nearby mobs while refreshing the ones we care
--   about.
--
--   Handles two possible workspace layouts:
--     A) Multiple Models named "Chief Petty Officer" directly under
--        Workspace.Enemies (standard Roblox spawning).
--     B) A Folder named "Chief Petty Officer" containing multiple enemy
--        Models (some games group same-type enemies in a sub-folder).
--
--   Returns: number of route-enemy NPCs found.
-- ============================================================================

function scanTargetEnemyFolder()
    if not LevelState.RouteEnemy or #LevelState.RouteEnemy == 0 then return 0 end

    local root = getLocalRoot()
    local rootPos = root and safeProp(root, "Position")
    if not rootPos then return 0 end

    local enemiesFolder = safeFind(Workspace, "Enemies")
    if not enemiesFolder then return 0 end

    local routeName   = LevelState.RouteEnemy
    local routeLower  = string_lower(routeName)
    local rangeSq     = Config.Range * Config.Range
    local found       = {}
    local seenKeys    = {}
    local activeHeads = {}

    -- Helper: try to add a model as an NPC entry
    local function tryAdd(model)
        if not model then return end
        local info = getNPCInfo(model)
        if info and not seenKeys[info.Key] then
            local d2 = distSq(rootPos, info.Pos)
            if d2 <= rangeSq then
                info.Distance = math_sqrt(d2)
                table_insert(found, info)
                seenKeys[info.Key] = true
                activeHeads[info.HeadKey] = true
                if Config.HeadHitbox then
                    applyHeadHitbox(info.Head, info.HeadKey)
                end
            end
        end
    end

    -- Strategy A: If Workspace.Enemies has a child named exactly like the
    -- route enemy AND it's a Folder, scan its children (layout B).
    -- If it's a Model, try adding it directly (layout A, first match).
    local namedChild = safeFind(enemiesFolder, routeName)
    if namedChild then
        local cn = safeProp(namedChild, "ClassName") or ""
        if cn == "Folder" then
            for _, m in ipairs(safeChildren(namedChild)) do
                tryAdd(m)
            end
        else
            tryAdd(namedChild)
        end
    end

    -- Strategy B: Also iterate ALL direct children of Workspace.Enemies and
    -- try adding any whose name matches the route enemy (case-insensitive).
    -- This catches duplicate-named Models (layout A with multiple instances)
    -- and is a no-op if Strategy A already covered them (seenKeys dedupes).
    for _, child in ipairs(safeChildren(enemiesFolder)) do
        local name = safeProp(child, "Name") or ""
        if string_lower(name) == routeLower then
            tryAdd(child)
        end
    end

    -- Merge: keep non-route-enemy NPCs from the existing NearbyNPCs list,
    -- replace route-enemy NPCs with the fresh scan.
    local merged = {}
    for _, npc in ipairs(NearbyNPCs) do
        if string_lower(npc.Name) ~= routeLower then
            table_insert(merged, npc)
        end
    end
    for _, npc in ipairs(found) do
        table_insert(merged, npc)
    end
    NearbyNPCs = merged

    -- Restore heads for NPCs that are no longer active (only among the
    -- route-enemy subset we just rescanned; non-route NPCs weren't touched).
    if Config.HeadHitbox then
        local allActiveHeads = {}
        for _, npc in ipairs(merged) do
            allActiveHeads[npc.HeadKey] = true
        end
        restoreInactiveHeads(allActiveHeads)
    end

    return #found
end

-- ============================================================================
-- 11. DRAWING HELPERS (ESP / health bars)
-- ============================================================================

function makeLabel()
    local t = Drawing_new("Text")
    t.Visible = false
    t.Center = true
    t.Outline = true
    t.Size = 14
    t.Color = Config.Accent
    t.ZIndex = 82
    pcall(function() t.Font = Drawing.Fonts.System end)
    return t
end

function getLabel(key)
    if not Labels[key] then Labels[key] = makeLabel() end
    return Labels[key]
end

function makeSquare(filled, z)
    local s = Drawing_new("Square")
    s.Visible = false
    s.Filled = filled
    s.ZIndex = z
    return s
end

function getBars(key)
    if not Bars[key] then
        Bars[key] = {
            Back   = makeSquare(true, 79),
            Fill   = makeSquare(true, 80),
            Border = makeSquare(false, 81),
        }
    end
    return Bars[key]
end

function setBarsVisible(group, visible)
    if not group then return end
    pcall(function() group.Back.Visible = visible end)
    pcall(function() group.Fill.Visible = visible end)
    pcall(function() group.Border.Visible = visible end)
end

function hideAllDrawings()
    for _, label in pairs(Labels) do pcall(function() label.Visible = false end) end
    for _, group in pairs(Bars) do setBarsVisible(group, false) end
    hideAllBoxes()
end

function removeAllDrawings()
    for _, label in pairs(Labels) do pcall(function() label:Remove() end) end
    for _, group in pairs(Bars) do
        pcall(function() group.Back:Remove() end)
        pcall(function() group.Fill:Remove() end)
        pcall(function() group.Border:Remove() end)
    end
    removeAllBoxes()
    Labels = {}
    Bars = {}
end

function clamp(v, a, b)
    if v > b then return b end
    if v < a then return a end
    return v
end

function healthColor(percent)
    if not Config.DynamicHealthColor then return Config.StaticHealthColor or COLOR_DEFAULT_HP end
    local p = clamp(percent, 0, 1)
    local r = math_floor(255 * (1 - p))
    local g = math_floor(255 * p)
    return Color3.fromRGB(r, g, 50)
end

function renderNpcOverlays()
    if not Config.NpcNametags and not Config.NpcHealthbars then
        hideAllDrawings()
        return
    end

    local root = getLocalRoot()
    local rootPos = root and safeProp(root, "Position")
    local used = {}
    local shown = 0

    if rootPos then
        local wantBox = Config.NpcBoxESP == true
        local wantHealth = Config.NpcHealthbars == true
        local wantNametag = Config.NpcNametags == true
        local yOffsetVec = Vector3_new(0, Config.YOffset, 0)
        local feetOffsetVec = Vector3_new(0, 5, 0)
        for _, npc in ipairs(NearbyNPCs) do
            if shown >= Config.MaxShown then break end
            -- Direct property access on cached Head instance is safe (we
            -- already validated it in getNPCInfo). pcall only as fallback.
            local pos
            local okPos = pcall(function() pos = npc.Head.Position end)
            if not okPos or not pos then
                pos = safeProp(npc.Head, "Position")
            end
            if pos then
                local drawPos = pos + yOffsetVec
                local ok, screen, onScreen = pcall(WorldToScreen, drawPos)
                if ok and screen and onScreen then
                    local x = math_floor(screen.X + 0.5)
                    local y = math_floor(screen.Y + 0.5)

                    -- Only project head+feet for Box ESP when BoxESP is ON.
                    -- Previously this ran 2 extra WorldToScreen calls per
                    -- NPC per frame even when BoxESP was off — wasteful.
                    local headScreen, feetScreen
                    if wantBox then
                        local okH, hs = pcall(WorldToScreen, pos)
                        if okH and hs then headScreen = hs end
                        local okF, fs = pcall(WorldToScreen, pos - feetOffsetVec)
                        if okF and fs then feetScreen = fs end
                    end

                    -- Re-read health live so the healthbar updates as the
                    -- mob takes damage (npc.Health was cached at scan time).
                    -- Direct access on the cached Humanoid is safe.
                    local hp = npc.Health
                    local maxHp = npc.MaxHealth
                    if wantHealth then
                        local hum = npc.Humanoid
                        if hum then
                            local okH, liveHp = pcall(function() return hum.Health end)
                            if okH and type(liveHp) == "number" then hp = liveHp end
                            local okM, liveMax = pcall(function() return hum.MaxHealth end)
                            if okM and type(liveMax) == "number" and liveMax > 0 then maxHp = liveMax end
                        end
                    end
                    local percent
                    if maxHp and maxHp > 0 then
                        percent = hp / maxHp
                    else
                        percent = 1
                    end
                    if percent < 0 then percent = 0 end
                    if percent > 1 then percent = 1 end

                    if wantNametag then
                        local label = getLabel(npc.Key)
                        local text = npc.Name
                        if Config.ShowDistance then
                            text = text .. " [" .. tostring(math_floor(npc.Distance + 0.5)) .. "]"
                        end
                        if Config.ShowHealthText then
                            text = text .. " " .. tostring(math_floor(percent * 100 + 0.5)) .. "%"
                        end
                        label.Text = text
                        label.Position = Vector2_new(x, y)
                        label.Color = Config.Accent
                        label.Visible = true
                    end

                    if wantHealth then
                        local group = getBars(npc.Key)
                        local w = Config.HealthbarWidth
                        local h = Config.HealthbarHeight
                        local bx = math_floor(x - w / 2)
                        local by = math_floor(y - 11)
                        local fillW = math_floor(w * percent)
                        if fillW < 1 then fillW = 1 end

                        group.Back.Position = Vector2_new(bx, by)
                        group.Back.Size = Vector2_new(w, h)
                        group.Back.Color = COLOR_HEALTHBAR_BG
                        group.Back.Visible = true

                        group.Fill.Position = Vector2_new(bx, by)
                        group.Fill.Size = Vector2_new(fillW, h)
                        group.Fill.Color = healthColor(percent)
                        group.Fill.Visible = true

                        group.Border.Position = Vector2_new(bx, by)
                        group.Border.Size = Vector2_new(w, h)
                        group.Border.Color = COLOR_BLACK
                        group.Border.Visible = true
                    end

                    -- Draw Box ESP around the ACTUAL NPC body (not the nametag)
                    if wantBox then
                        if headScreen and feetScreen then
                            local boxLines = getBoxLines(npc.Key)
                            -- Calculate box bounds from head and feet screen positions
                            local headY = math_floor(headScreen.Y + 0.5)
                            local feetY = math_floor(feetScreen.Y + 0.5)
                            local centerX = math_floor(headScreen.X + 0.5)
                            -- Box width: proportional to height (roughly 40% of height)
                            local boxHeight = feetY - headY
                            if boxHeight < 10 then boxHeight = 30 end  -- fallback if too small
                            local boxWidth = math_floor(boxHeight * 0.5)
                            if boxWidth < 8 then boxWidth = 15 end
                            -- Extend top a bit above head (for head hitbox)
                            local topY = headY - math_floor(boxHeight * 0.2)
                            local botY = feetY
                            local leftX = centerX - boxWidth
                            local rightX = centerX + boxWidth

                            local boxColor = isBossEnemy(npc.Name) and COLOR_BOSS_RED or COLOR_WHITE
                            -- Top
                            boxLines[1].From = Vector2_new(leftX, topY)
                            boxLines[1].To   = Vector2_new(rightX, topY)
                            boxLines[1].Color = boxColor
                            boxLines[1].Visible = true
                            -- Bottom
                            boxLines[2].From = Vector2_new(leftX, botY)
                            boxLines[2].To   = Vector2_new(rightX, botY)
                            boxLines[2].Color = boxColor
                            boxLines[2].Visible = true
                            -- Left
                            boxLines[3].From = Vector2_new(leftX, topY)
                            boxLines[3].To   = Vector2_new(leftX, botY)
                            boxLines[3].Color = boxColor
                            boxLines[3].Visible = true
                            -- Right
                            boxLines[4].From = Vector2_new(rightX, topY)
                            boxLines[4].To   = Vector2_new(rightX, botY)
                            boxLines[4].Color = boxColor
                            boxLines[4].Visible = true
                        end
                    end

                    used[npc.Key] = true
                    shown = shown + 1
                end
            end
        end
    end

    for key, label in pairs(Labels) do
        if not used[key] then pcall(function() label.Visible = false end) end
    end
    for key, group in pairs(Bars) do
        if not used[key] then setBarsVisible(group, false) end
    end
    for key, lines in pairs(Boxes) do
        if not used[key] then
            for _, l in ipairs(lines) do pcall(function() l.Visible = false end) end
        end
    end
end

-- ============================================================================
-- 12. TWEENING (custom, since Matcha has no TweenInfo.new)
-- ============================================================================

function clampNumber(value, low, high)
    if value < low then return low end
    if value > high then return high end
    return value
end

function lerpVector3(a, b, alpha)
    return Vector3_new(a.X + (b.X - a.X) * alpha,
                       a.Y + (b.Y - a.Y) * alpha,
                       a.Z + (b.Z - a.Z) * alpha)
end

function easingAlpha(style, alpha)
    if style == "linear" then
        return alpha
    elseif style == "ease_in_quad" then
        return alpha * alpha
    elseif style == "ease_out_quad" then
        return alpha * (2 - alpha)
    elseif style == "ease_in_out_quad" then
        if alpha < 0.5 then return 2 * alpha * alpha end
        return -1 + (4 - 2 * alpha) * alpha
    end
    -- smoothstep (default)
    return alpha * alpha * (3 - 2 * alpha)
end

-- Tween root to a target position, looking at lookPosition.
-- Returns a signal table { Completed = bool, Cancelled = bool } that the caller polls.
-- Features:
--   - Uses RenderStepped for smooth frame-synced movement (falls back to task_wait)
--   - Cancels if root position is nil or root is destroyed
--   - Clamps to 60Hz max update rate (avoids overloading Matcha)
--   - Sets velocity to zero on completion to prevent drift
--   - Supports instant teleport for very short distances (< 5 studs)
function tweenRootTo(root, targetPosition, lookPosition, speed, easingStyle)
    local result = { Completed = false, Cancelled = false }
    local tweenSpeed  = speed or Config.LevelTweenSpeed
    local tweenEasing = easingStyle or Config.LevelTweenEasing

    if not root or not targetPosition or tweenSpeed <= 0 then
        result.Completed = true
        return result
    end

    local startPosition = safeProp(root, "Position")
    if not startPosition then
        result.Completed = true
        return result
    end

    local distance = math_sqrt(distSq(startPosition, targetPosition))

    -- Instant teleport for very short distances
    if distance < 5 then
        pcall(function()
            root.CFrame = CFrame_lookAt(targetPosition, lookPosition or targetPosition)
            root.AssemblyLinearVelocity = Vector3_new(0, 0, 0)
        end)
        result.Completed = true
        return result
    end

    -- Cap the tween speed for long distances to prevent rubberbanding.
    -- Roblox server validation rejects movement that's too fast — if you
    -- travel 2000+ studs at 5000 studs/s, the server thinks you're
    -- teleport-hacking and rubberbands you back.
    -- For distances > 500 studs, cap speed to 300 studs/s (safe long-range).
    -- For distances > 2000 studs, cap speed to 200 studs/s (very safe).
    local effectiveSpeed = tweenSpeed
    if distance > 2000 then
        effectiveSpeed = math.min(tweenSpeed, 200)
    elseif distance > 500 then
        effectiveSpeed = math.min(tweenSpeed, 300)
    end

    local duration = distance / effectiveSpeed
    if duration <= 0 then
        result.Completed = true
        return result
    end

    task_spawn(function()
        local elapsed = 0
        local lastTime = os_clock()
        local useRenderStepped = false

        -- Try to use RenderStepped for smooth movement
        local RS = game:GetService("RunService")
        local conn
        pcall(function()
            if RS and RS.RenderStepped then
                useRenderStepped = true
            end
        end)

        if useRenderStepped then
            -- Frame-synced tween via RenderStepped
            local function onStep(dt)
                if not Running or result.Cancelled then
                    if conn then pcall(function() conn:Disconnect() end) end
                    return true
                end
                elapsed = elapsed + (dt or 0.016)
                local alpha = clampNumber(elapsed / duration, 0, 1)
                local eased = easingAlpha(tweenEasing, alpha)
                local nextPosition = lerpVector3(startPosition, targetPosition, eased)

                local ok = pcall(function()
                    root.CFrame = CFrame_lookAt(nextPosition, lookPosition or targetPosition)
                end)
                if not ok then
                    result.Cancelled = true
                    if conn then pcall(function() conn:Disconnect() end) end
                    return true
                end

                if alpha >= 1 then
                    pcall(function()
                        root.CFrame = CFrame_lookAt(targetPosition, lookPosition or targetPosition)
                        root.AssemblyLinearVelocity = Vector3_new(0, 0, 0)
                    end)
                    result.Completed = true
                    if conn then pcall(function() conn:Disconnect() end) end
                    return true
                end
                return false
            end

            -- Connect and run
            pcall(function()
                conn = RS.RenderStepped:Connect(function(dt)
                    onStep(dt)
                end)
            end)

            -- Fallback: if RenderStepped connection failed, use task_wait
            if not conn then
                useRenderStepped = false
            end
        end

        if not useRenderStepped then
            -- Fallback: task_wait-based tween (60Hz)
            local dt = 1 / 60
            while Running and elapsed < duration do
                elapsed = elapsed + dt
                local alpha = clampNumber(elapsed / duration, 0, 1)
                local eased = easingAlpha(tweenEasing, alpha)
                local nextPosition = lerpVector3(startPosition, targetPosition, eased)
                local ok = pcall(function()
                    root.CFrame = CFrame_lookAt(nextPosition, lookPosition or targetPosition)
                end)
                if not ok then
                    result.Cancelled = true
                    break
                end
                task_wait(dt)
            end
            if not result.Cancelled then
                pcall(function()
                    root.CFrame = CFrame_lookAt(targetPosition, lookPosition or targetPosition)
                    root.AssemblyLinearVelocity = Vector3_new(0, 0, 0)
                end)
                result.Completed = true
            end
        end
    end)

    return result
end

function getStandPositionNear(targetPos, distance, heightOffset)
    local root = getLocalRoot()
    local rootPos = root and safeProp(root, "Position")
    if not root or not rootPos or not targetPos then return nil, nil, nil end
    local dx = rootPos.X - targetPos.X
    local dz = rootPos.Z - targetPos.Z
    local flatLen = math_sqrt(dx * dx + dz * dz)
    if flatLen < 1 then dx = 0; dz = 1; flatLen = 1 end
    local standPos = Vector3_new(
        targetPos.X + (dx / flatLen) * distance,
        targetPos.Y + heightOffset,
        targetPos.Z + (dz / flatLen) * distance
    )
    return root, standPos, targetPos
end

function getLevelStandPosition(npc)
    local targetPos = npc and (safeProp(npc.Head, "Position") or npc.Pos)
    return getStandPositionNear(targetPos, Config.LevelDistance, Config.LevelHeightOffset)
end

function moveToLevelTarget(npc)
    if not Config.LevelMoveToTarget then return false end
    if not npc then return false end
    if LevelState.TweenSignal and not LevelState.TweenSignal.Completed then return true end
    local root, standPos, targetPos = getLevelStandPosition(npc)
    if not root or not standPos or not targetPos then return false end
    LevelState.TweenSignal  = tweenRootTo(root, standPos, targetPos)
    LevelState.TweenPurpose = "enemy"
    return true
end

function tweenToWaypoint(name, pos)
    local root = getLocalRoot()
    if not root or not pos then
        notifyUser("Teleport unavailable: no root", 4)
        return false
    end
    local targetPos = pos + Vector3_new(0, Config.TeleportHeightOffset, 0)
    LevelState.TweenSignal  = tweenRootTo(root, targetPos, pos, Config.TeleportTweenSpeed, Config.LevelTweenEasing)
    LevelState.TweenPurpose = "teleport"
    notifyUser("Tweening to " .. name, 4)
    return true
end

-- ============================================================================
-- 13. INPUT HELPERS  (Matcha-specific — 3-arg mousemoveabs!)
-- ============================================================================

-- Try multiple mousemoveabs signatures. The Matcha docs say `mousemoveabs(0, x, y)`
-- requires a leading 0, but in practice this seems to move the mouse to (0, x)
-- instead of (x, y) — putting it in the bottom-left corner.
-- We try (x, y) first, then fall back to (0, x, y) if that throws.
local _mousemoveabs_variant = nil  -- cached: "noextra" | "leading0" | "trailing0"

safeMouseMoveAbs = function(x, y)
    if type(mousemoveabs) ~= "function" then return false end
    x = math_floor(x)
    y = math_floor(y)

    -- Use cached variant if we've already detected one that works
    if _mousemoveabs_variant == "noextra" then
        local ok = pcall(function() mousemoveabs(x, y) end)
        if ok then return true end
        _mousemoveabs_variant = nil  -- cache miss, re-detect
    elseif _mousemoveabs_variant == "leading0" then
        local ok = pcall(function() mousemoveabs(0, x, y) end)
        if ok then return true end
        _mousemoveabs_variant = nil
    elseif _mousemoveabs_variant == "trailing0" then
        local ok = pcall(function() mousemoveabs(x, y, 0) end)
        if ok then return true end
        _mousemoveabs_variant = nil
    end

    -- Detection: try each variant in pcall. Pick the first that doesn't throw.
    -- Prefer (x, y) first since that's the standard signature and avoids the
    -- "leading 0 = x coord" misinterpretation issue.
    if pcall(function() mousemoveabs(x, y) end) then
        _mousemoveabs_variant = "noextra"
        qlog(string_format("safeMouseMoveAbs: using variant 'noextra' (mousemoveabs(x, y)) for (%d, %d)", x, y))
        return true
    end
    if pcall(function() mousemoveabs(0, x, y) end) then
        _mousemoveabs_variant = "leading0"
        qlog(string_format("safeMouseMoveAbs: using variant 'leading0' (mousemoveabs(0, x, y)) for (%d, %d)", x, y))
        return true
    end
    if pcall(function() mousemoveabs(x, y, 0) end) then
        _mousemoveabs_variant = "trailing0"
        qlog(string_format("safeMouseMoveAbs: using variant 'trailing0' (mousemoveabs(x, y, 0)) for (%d, %d)", x, y))
        return true
    end

    qlog("safeMouseMoveAbs: ALL variants failed for (" .. tostring(x) .. ", " .. tostring(y) .. ")")
    return false
end

safeMouse1Click = function()
    if type(mouse1click) == "function" then
        return pcall(function() mouse1click() end)
    end
    -- fallback: press/release pair
    if type(mouse1press) == "function" and type(mouse1release) == "function" then
        local ok1 = pcall(function() mouse1press() end)
        task_wait(Config.LevelClickHold)
        local ok2 = pcall(function() mouse1release() end)
        return ok1 and ok2
    end
    return false
end

-- "Hold click" pattern: press, wait, release. More reliable for GUI buttons
-- than a single mouse1click (which may be too fast for Roblox to register).
safeMouse1HoldClick = function(holdTime)
    holdTime = holdTime or 0.05
    if type(mouse1press) == "function" and type(mouse1release) == "function" then
        local ok1 = pcall(function() mouse1press() end)
        task_wait(holdTime)
        local ok2 = pcall(function() mouse1release() end)
        return ok1 and ok2
    end
    -- fallback: single click
    if type(mouse1click) == "function" then
        return pcall(function() mouse1click() end)
    end
    return false
end

safeMouse1Press = function()
    if type(mouse1press) == "function" then
        return pcall(function() mouse1press() end)
    end
    return false
end

safeMouse1Release = function()
    if type(mouse1release) == "function" then
        return pcall(function() mouse1release() end)
    end
    return false
end

-- CRITICAL: We do NOT call setrobloxinput() in guiClickAt.
-- Per Matcha docs: "setrobloxinput(...) exists; tested but purpose unknown.
-- Does NOT block InputBegan, iskeypressed, or change isrbxactive(). Avoid
-- calling with anything other than true/false."
--
-- User reports that calling setrobloxinput makes the cursor "turn into a
-- regular mouse instead of the roblox mouse" — confirming that toggling it
-- mid-frame disrupts Roblox's mouse capture. The UILib already manages
-- setrobloxinput correctly (false when menu open, true when closed), so we
-- let it handle it. mouse1click + mousemoveabs reach GUI buttons fine on
-- their own — no input toggling needed.
local _robloxInputSavedState = nil
enableRobloxInput = function()
    -- No-op: intentionally do nothing. UILib:Step() manages setrobloxinput.
end
restoreRobloxInput = function()
    -- No-op: intentionally do nothing.
end

-- Full GUI button click sequence:
--   1. Move mouse to (x, y)
--   2. Wait for mouse to settle
--   3. BURST CLICK 5 times — ensures the button gets pressed even if the
--      first few clicks miss due to timing/rendering issues
guiClickAt = function(x, y, opts)
    opts = opts or {}
    local holdTime = opts.holdTime or 0.04
    local settleTime = opts.settleTime or Config.QuestUiDelay
    local burstCount = opts.burstCount or 5

    if not isValidScreenCoord(x, y) then
        qlog(string_format("guiClickAt REJECTED bogus coord (%s, %s)",
            tostring(x), tostring(y)))
        return false
    end

    -- Step 1: move mouse to target
    safeMouseMoveAbs(x, y)
    task_wait(settleTime)

    -- Step 2: BURST CLICK with mouse jitter between each click
    -- Jitter: move mouse ±2px randomly between clicks so Roblox registers
    -- mouse movement (Blox Fruits needs this to register GUI button clicks).
    for i = 1, burstCount do
        -- Jitter the mouse slightly before each click
        local jx = x + math_random(-2, 2)
        local jy = y + math_random(-2, 2)
        safeMouseMoveAbs(jx, jy)
        task_wait(0.02)

        if type(mouse1press) == "function" and type(mouse1release) == "function" then
            pcall(function() mouse1press() end)
            task_wait(holdTime)
            pcall(function() mouse1release() end)
            task_wait(0.02)
        elseif type(mouse1click) == "function" then
            pcall(function() mouse1click() end)
            task_wait(0.03)
        end
    end

    return true
end

-- We do NOT call setrobloxinput() at all in this script.
-- Per Matcha docs: it doesn't do what people think it does — it doesn't block
-- iskeypressed, doesn't change isrbxactive(), and the only safe usage pattern
-- is the UILib's own internal calls. Leaving it alone.

-- (Forward declarations are at the top of the file, section 2.5)

-- Validate a screen position is within the viewport and NOT at the extreme
-- corners (which is where WorldToScreen dumps garbage for off-screen points).
-- Returns true if the position is safe to click.
isValidScreenCoord = function(x, y)
    if type(x) ~= "number" or type(y) ~= "number" then return false end
    if x < 0 or y < 0 then return false end
    local vpX, vpY = getViewportSize()
    -- Reject positions within 20px of any edge (WorldToScreen returns these
    -- for points that project off-screen but technically pass onScreen=true)
    if x < 20 or y < 20 then return false end
    if x > vpX - 20 or y > vpY - 20 then return false end
    return true
end

clickScreenAt = function(x, y)
    if not isValidScreenCoord(x, y) then
        qlog(string_format("clickScreenAt REJECTED bogus coord (%s, %s)",
            tostring(x), tostring(y)))
        return false
    end
    safeMouseMoveAbs(x, y)
    task_wait(Config.QuestUiDelay)
    return safeMouse1Click()
end

clickWorldPosition = function(pos)
    if not pos then return false end
    local ok, screen, onScreen = pcall(WorldToScreen, pos)
    if not (ok and screen and onScreen) then
        qlog("clickWorldPosition: WorldToScreen returned off-screen for " .. tostring(pos))
        return false
    end
    -- Extra safety: reject bogus WorldToScreen results
    if not isValidScreenCoord(screen.X, screen.Y) then
        qlog(string_format("clickWorldPosition: bogus screen coord (%.1f, %.1f) for world pos %s",
            screen.X, screen.Y, tostring(pos)))
        return false
    end
    -- Use guiClickAt for reliable clicks (enables Roblox input, hold-click pattern)
    return guiClickAt(screen.X, screen.Y, { holdTime = 0.08, settleTime = Config.QuestUiDelay })
end

-- ============================================================================
-- 14. CLICK MANAGEMENT (for autoclicker)
-- ============================================================================

function releaseClickIfNeeded()
    if not LevelState.ClickHeld then return end
    if os_clock() < LevelState.ClickReleaseAt then return end
    safeMouse1Release()
    LevelState.ClickHeld = false
end

function forceReleaseClick()
    safeMouse1Release()
    LevelState.ClickHeld = false
end

function clickLevelTarget()
    if not Config.LevelAutoClick then return false end
    forceReleaseClick()

    local now = os_clock()
    if now - LevelState.LastClickAt < Config.LevelClickInterval then return false end

    -- Move the mouse to the center of the screen first (where the mob should be
    -- since we're hovering above it looking down). This ensures the click
    -- lands on the mob in Roblox's view.
    local vpX, vpY = getViewportSize()
    local cx, cy = math_floor(vpX / 2), math_floor(vpY / 2)
    safeMouseMoveAbs(cx, cy)
    task_wait(0.02)

    -- Burst click 5 times at the center of the screen
    for i = 1, 5 do
        if type(mouse1press) == "function" and type(mouse1release) == "function" then
            pcall(function() mouse1press() end)
            task_wait(0.04)
            pcall(function() mouse1release() end)
            task_wait(0.02)
        elseif type(mouse1click) == "function" then
            pcall(function() mouse1click() end)
            task_wait(0.03)
        end
    end

    LevelState.LastClickAt = now
    return true
end

-- ============================================================================
-- 15. QUEST GIVER FINDER
-- ============================================================================

function trimText(text)
    return string.gsub(tostring(text or ""), "^%s*(.-)%s*$", "%1")
end

-- Helper: returns true if `haystack` contains `needle` as a WHOLE WORD.
-- This prevents "Marine" from matching "Marines" (the s makes it a different
-- word) while still matching "Marine Quest Giver" (where Marine is a token).
-- Word boundaries are: start of string, end of string, or any non-alphanumeric
-- character (space, punctuation).
local function containsWord(haystack, needle)
    if #needle == 0 or #haystack < #needle then return false end
    -- Pattern: boundary, literal needle, boundary
    -- Lua patterns don't support \b, so we use frontier %f[%W] etc.
    -- Simpler: scan all positions and check the chars before/after.
    local start = 1
    while true do
        local i, j = string_find(haystack, needle, start, true)
        if not i then return false end
        local before = i == 1 and "" or string_sub(haystack, i - 1, i - 1)
        local after  = j == #haystack and "" or string_sub(haystack, j + 1, j + 1)
        local beforeIsBoundary = (before == "" or string_match(before, "[%W_]") ~= nil)
        local afterIsBoundary  = (after == "" or string_match(after, "[%W_]") ~= nil)
        if beforeIsBoundary and afterIsBoundary then
            return true
        end
        start = i + 1
    end
end

-- Check if a found object's name matches the wanted quest giver name.
-- The game uses SHORTER names than our QuestRoutes table:
--   QuestRoutes says "Jungle Adventurer" but the NPC is actually named "Adventurer"
--   QuestRoutes says "The Mayor" but the NPC is actually named "Mayor"
-- So we need loose matching, BUT with WORD-BOUNDARY enforcement to prevent
-- "Marine" from matching "Advanced Marines Boat Dealer" (since "Marine" is
-- a substring of "Marines" but NOT a separate word).
--
-- Matching tiers (in order, first match wins):
--   1. Exact match (case-insensitive): "marine" == "marine"
--   2. Found name contains query as a whole word:
--        "marine quest giver" contains word "marine"  ✓
--   3. Query contains found name as a whole word (with min-length 4 guard):
--        "the mayor" contains word "mayor"  ✓
--        (the min-length guard prevents matching "The", "Sky", etc.)
--
-- Position validation in findQuestGiver (must be within 500 studs of the
-- expected position from QuestRoutes) is the secondary filter.
function nameMatchesQuestGiver(name, query)
    local lowerName = string_lower(tostring(name or ""))
    if #lowerName < 1 then return false end
    for part in string.gmatch(tostring(query or ""), "([^/]+)") do
        local candidate = string_lower(trimText(part))
        if #candidate > 0 then
            -- Tier 1: Exact match
            if lowerName == candidate then return true end
            -- Tier 2: Found name contains query as a whole word
            if containsWord(lowerName, candidate) then return true end
            -- Tier 3: Query contains found name as a whole word
            -- Only allow if found name is at least 4 chars (avoids matching
            -- "The", "Sky", etc.)
            if #lowerName >= 4 and containsWord(candidate, lowerName) then
                return true
            end
        end
    end
    return false
end

function getWantedQuestGiver()
    if type(Config.QuestGiverOverride) == "string" and #Config.QuestGiverOverride > 0 then
        return Config.QuestGiverOverride
    end
    return LevelState.RouteQuestGiver
end

function getObjectPosition(obj)
    if not obj then return nil end
    local pos = safeProp(obj, "Position")
    if pos then return pos end
    local primary = safeProp(obj, "PrimaryPart")
    pos = primary and safeProp(primary, "Position")
    if pos then return pos end
    local root = safeFind(obj, "HumanoidRootPart") or safeFind(obj, "Head") or safeFind(obj, "Torso") or safeFind(obj, "UpperTorso")
    pos = root and safeProp(root, "Position")
    if pos then return pos end
    for _, child in ipairs(safeChildren(obj)) do
        pos = safeProp(child, "Position")
        if pos then return pos end
    end
    return nil
end

-- Fast quest-giver finder: only check workspace.NPCs first (where Blox Fruits
-- puts them), then fall back to a broader walk if needed.
function findQuestGiver()
    local wanted = getWantedQuestGiver()
    if type(wanted) ~= "string" or #wanted < 1 or wanted == "none" then
        return nil, nil
    end

    -- CRITICAL: We have the expected giver position from QuestRoutes.
    -- Use it to validate any found giver — must be within 500 studs of
    -- the expected position. This prevents finding random objects with
    -- similar names far away (e.g. a tree named "Jungle" in the ocean).
    local expectedPos = LevelState.RouteQuestGiverCFrame
    local function isValidGiverPosition(pos)
        if not pos then return false end
        if not expectedPos then return true end  -- no expected position, accept anything
        local d = math_sqrt(distSq(pos, expectedPos))
        if d > 500 then
            qlog(string_format("findQuestGiver: rejecting giver at %s — %.0f studs from expected position %s (too far)",
                tostring(pos), d, tostring(expectedPos)))
            return false
        end
        return true
    end

    -- Helper: returns true if `child` is a valid quest giver candidate.
    -- Combines name match + shop NPC blacklist check.
    -- Logs rejections so we can debug "why didn't it pick X".
    local function isCandidate(child, name)
        if not nameMatchesQuestGiver(name, wanted) then return false end
        if isShopNpc(name) then
            qlog(string_format("findQuestGiver: rejecting '%s' — matches wanted '%s' but is a SHOP NPC (boat dealer / weaponsmith / etc.)",
                tostring(name), tostring(wanted)))
            return false
        end
        return true
    end

    -- Priority 1: workspace.NPCs (Blox Fruits' actual quest giver folder)
    local npcsFolder = safeFind(Workspace, "NPCs")
    if npcsFolder then
        for _, child in ipairs(safeChildren(npcsFolder)) do
            local name = safeProp(child, "Name")
            if isCandidate(child, name) then
                local pos = getObjectPosition(child)
                if pos and isValidGiverPosition(pos) then return child, pos end
            end
            -- one level of nesting
            for _, grand in ipairs(safeChildren(child)) do
                local gname = safeProp(grand, "Name")
                if isCandidate(grand, gname) then
                    local gpos = getObjectPosition(grand)
                    if gpos and isValidGiverPosition(gpos) then return grand, gpos end
                end
            end
        end
    end

    -- Priority 2: known subfolders
    local priorityFolders = {"QuestGivers", "Quest Givers", "Quests", "Characters", "Map", "World"}
    for _, folderName in ipairs(priorityFolders) do
        local folder = safeFind(Workspace, folderName)
        if folder then
            for _, child in ipairs(safeChildren(folder)) do
                local name = safeProp(child, "Name")
                if isCandidate(child, name) then
                    local pos = getObjectPosition(child)
                    if pos and isValidGiverPosition(pos) then return child, pos end
                end
            end
        end
    end

    -- Priority 3: bounded workspace walk (last resort)
    local visited = 0
    local maxVisited = 3000
    local maxDepth = 6
    local function walk(obj, depth)
        if not obj or depth > maxDepth or visited > maxVisited then return nil, nil end
        visited = visited + 1
        local name = safeProp(obj, "Name")
        if isCandidate(obj, name) then
            local pos = getObjectPosition(obj)
            if pos and isValidGiverPosition(pos) then return obj, pos end
        end
        for _, child in ipairs(safeChildren(obj)) do
            local found, pos = walk(child, depth + 1)
            if found and pos then return found, pos end
        end
        return nil, nil
    end
    return walk(Workspace, 0)
end

-- ============================================================================
-- 16. QUEST GUI DETECTION
--    Dialog lives at Players.LocalPlayer.PlayerGui.Main.Dialogue
--    Active quest tracker lives at Players.LocalPlayer.PlayerGui.Main.Quest
--    The quest title (with kill counter) is at:
--       PlayerGui.Main.Quest.Container.QuestTitle.Title.Text
-- ============================================================================

local QuestConfirmLabels = {"Confirm", "Accept", "Yes", "Start Quest", "Start", "Begin Quest"}

getQuestGuiRoots = function()
    local roots = {}
    local lp = getLocalPlayer()
    if lp then
        local pg = safeFind(lp, "PlayerGui")
        if pg then table.insert(roots, pg) end
    end
    pcall(function()
        local cg = game:GetService("CoreGui")
        if cg then table.insert(roots, cg) end
    end)
    pcall(function()
        local sg = game:GetService("StarterGui")
        if sg then table.insert(roots, sg) end
    end)
    return roots
end

numberField = function(value, upperName, lowerName)
    if not value then return nil end
    local result = safeProp(value, upperName)
    if type(result) == "number" then return result end
    result = safeProp(value, lowerName)
    if type(result) == "number" then return result end
    return nil
end

-- Assign to the forward-declared local (no `local` keyword here)
getViewportSize = function()
    local x, y = 1920, 1080
    pcall(function()
        local camera = Workspace and Workspace.CurrentCamera
        local viewport = camera and camera.ViewportSize
        local vx = numberField(viewport, "X", "x")
        local vy = numberField(viewport, "Y", "y")
        if vx and vy then x = vx; y = vy end
    end)
    return x, y
end

getGuiCenter = function(obj)
    local visible = safeProp(obj, "Visible")
    if visible == false then return nil, nil end
    local position = safeProp(obj, "AbsolutePosition")
    local size = safeProp(obj, "AbsoluteSize")
    local x = numberField(position, "X", "x")
    local y = numberField(position, "Y", "y")
    local width = numberField(size, "X", "x")
    local height = numberField(size, "Y", "y")
    if not x or not y or not width or not height then return nil, nil end
    if width < 1 or height < 1 then return nil, nil end
    return math_floor(x + width / 2), math_floor(y + height / 2)
end

-- Returns true when obj is a TextButton (NOT TextLabel — labels are not
-- clickable) with non-empty Text, visible, NOT inside a BillboardGui/SurfaceGui,
-- and has a non-trivial AbsoluteSize.
-- CRITICAL: We require TextButton specifically because TextLabels are used
-- for dialog titles like "QUEST" and would otherwise get picked as candidates.
isValidQuestButton = function(obj)
    if not obj then return false end
    local className = string_lower(safeProp(obj, "ClassName") or "")
    -- ONLY TextButton — TextLabel is for titles/labels, not clickable
    if className ~= "textbutton" then return false end

    -- Must have non-empty Text
    local text = safeProp(obj, "Text")
    if type(text) ~= "string" or #text < 1 then return false end

    -- Must have a non-trivial AbsoluteSize
    local size = safeProp(obj, "AbsoluteSize")
    local w = numberField(size, "X", "x")
    local h = numberField(size, "Y", "y")
    if not w or not h or w < 4 or h < 4 then return false end

    -- Walk upward: every ancestor must be visible, no world-space containers.
    local current = obj
    local depth = 0
    while current and depth < 30 do
        local cn = string_lower(safeProp(current, "ClassName") or "")
        if cn == "billboardgui" or cn == "surfacegui" then
            return false
        end
        local visible = safeProp(current, "Visible")
        if visible == false then return false end
        local parent = safeProp(current, "Parent")
        if not parent then break end
        local parentClass = string_lower(safeProp(parent, "ClassName") or "")
        if parentClass == "datamodel" then break end
        current = parent
        depth = depth + 1
    end
    return true
end

cleanQuestText = function(value)
    if type(value) ~= "string" then return "" end
    local text = string_lower(value)
    text = text:gsub("%b[]", " ")
    text = text:gsub("[\r\n\t]+", " ")
    text = text:gsub("[^%w%s%-%']", " ")
    text = text:gsub("%s+", " ")
    text = text:gsub("^%s+", "")
    text = text:gsub("%s+$", "")
    return text
end

compactQuestText = function(value)
    local text = cleanQuestText(value)
    text = text:gsub("[%-%']", "")
    text = text:gsub("%s+", " ")
    return text
end

pluralQuestText = function(value)
    if #value < 1 then return value end
    local last = value:sub(#value, #value)
    if last == "s" then return value end
    if last == "y" then
        return value:sub(1, #value - 1) .. "ies"
    end
    return value .. "s"
end

questTextMatches = function(text, wanted)
    local cleanText = cleanQuestText(text)
    local cleanWanted = cleanQuestText(wanted)
    if #cleanText < 1 or #cleanWanted < 1 then return false end
    if cleanText == cleanWanted then return true end
    if cleanText == pluralQuestText(cleanWanted) then return true end
    if cleanText:sub(#cleanText, #cleanText) == "s" and cleanText:sub(1, #cleanText - 1) == cleanWanted then return true end
    local compactText = compactQuestText(text)
    local compactWanted = compactQuestText(wanted)
    if compactText == compactWanted then return true end
    if compactText == pluralQuestText(compactWanted) then return true end
    if compactText:sub(#compactText, #compactText) == "s" and compactText:sub(1, #compactText - 1) == compactWanted then return true end
    -- NOTE: We DO NOT do bidirectional substring match anymore.
    -- That caused false positives like "Gorilla King" matching "Gorilla".
    -- Option buttons must match the enemy name exactly (singular or plural).
    return false
end

-- Strict match for option buttons: text must equal the enemy name (singular
-- or plural). No substring matches. Rejects "Gorilla King" when looking
-- for "Gorilla".
questOptionTextMatches = function(text, wanted)
    local cleanText = cleanQuestText(text)
    local cleanWanted = cleanQuestText(wanted)
    if #cleanText < 1 or #cleanWanted < 1 then return false end
    if cleanText == cleanWanted then return true end
    if cleanText == pluralQuestText(cleanWanted) then return true end
    if cleanText:sub(#cleanText, #cleanText) == "s" and cleanText:sub(1, #cleanText - 1) == cleanWanted then return true end
    local compactText = compactQuestText(text)
    local compactWanted = compactQuestText(wanted)
    if compactText == compactWanted then return true end
    if compactText == pluralQuestText(compactWanted) then return true end
    if compactText:sub(#compactText, #compactText) == "s" and compactText:sub(1, #compactText - 1) == compactWanted then return true end
    return false
end

-- Strict match for confirm buttons: text must EXACTLY equal one of the known
-- confirm labels (case-insensitive, after cleaning).
questConfirmTextMatches = function(text)
    if type(text) ~= "string" then return false end
    local cleanText = cleanQuestText(text)
    if #cleanText < 1 then return false end
    for _, label in ipairs(QuestConfirmLabels) do
        local cleanLabel = cleanQuestText(label)
        if cleanText == cleanLabel then return true end
    end
    return false
end

scoreQuestGuiCandidate = function(obj, x, y)
    if not isValidQuestButton(obj) then return -99999 end

    local score = 0
    local name = string_lower(safeProp(obj, "Name") or "")
    local fullName = string_lower(safeFullName(obj))
    local text = string_lower(safeProp(obj, "Text") or "")
    local vpX, vpY = getViewportSize()

    if string_find(name, "quest", 1, true)    then score = score + 40 end
    if string_find(fullName, "quest", 1, true)then score = score + 40 end
    if string_find(name, "confirm", 1, true)  then score = score + 60 end
    if string_find(name, "accept", 1, true)   then score = score + 60 end

    -- Strongest signal: exact text match on a known confirm label
    if text == "confirm" or text == "accept" or text == "yes" or text == "start quest" then
        score = score + 300
    end

    -- Penalize obvious cancel/close/nevermind AND non-quest action buttons
    -- (Stats/Items/Close/Shop are present in the Blox Fruits quest dialog
    -- but they are NOT quest option buttons — they open other menus)
    local rejectPatterns = {
        "return", "nevermind", "never mind", "cancel", "close", "exit",
        "leave", "back", "stats", "items", "shop", "menu", "settings",
        "inventory", "help", "ok",
    }
    for _, pat in ipairs(rejectPatterns) do
        if string_find(text, pat, 1, true) then
            score = score - 500
            break
        end
    end

    -- HARD REJECT: title text at the very top of the screen (y < 15% of viewport)
    -- These are dialog titles like "QUEST" or "Adventurer" labels
    if y < vpY * 0.15 then
        score = score - 800
    end

    -- Penalize far-left position (where Stats/Items/Close/Shop buttons live)
    if x < vpX * 0.25 then
        score = score - 300
    end

    -- Penalize extreme top/bottom
    if y < vpY * 0.10 or y > vpY * 0.90 then
        score = score - 200
    end
    -- Mild bonus for being in the expected vertical band (dialog option buttons)
    if y > vpY * 0.20 and y < vpY * 0.85 then
        score = score + 20
    end
    return score
end

-- Find a visible TextButton whose Text matches wantedText.
-- Uses STRICT matching (no substring) so "Gorilla King" doesn't match "Gorilla".
-- Returns x, y, obj — or nil, nil, nil if nothing found.
findQuestGuiButton = function(wantedText)
    if not Config.QuestGuiDetect then return nil, nil, nil end
    if type(wantedText) ~= "string" or #wantedText < 1 then return nil, nil, nil end

    local bestObj, bestX, bestY, bestScore = nil, nil, nil, -99999
    local candidates = {}  -- for diagnostic logging
    local roots = getQuestGuiRoots()

    for _, root in ipairs(roots) do
        for _, obj in ipairs(safeDescendants(root)) do
            local text = safeProp(obj, "Text")
            if type(text) == "string" and #text > 0 and questOptionTextMatches(text, wantedText) then
                local x, y = getGuiCenter(obj)
                if x and y then
                    local score = scoreQuestGuiCandidate(obj, x, y)
                    table.insert(candidates, {
                        text = text:sub(1, 40), x = x, y = y, score = score,
                        path = safeFullName(obj):sub(1, 80),
                    })
                    if score > bestScore then
                        bestObj, bestX, bestY, bestScore = obj, x, y, score
                    end
                end
            end
        end
    end

    -- Log all candidates when there are multiple, so we can see what's being picked
    if #candidates >= 1 then
        qlog(string_format("findQuestGuiButton(%q) found %d candidate(s):",
            tostring(wantedText), #candidates))
        for _, c in ipairs(candidates) do
            qlog(string_format("  score=%d text=%q pos=(%d,%d) path=%s",
                c.score, c.text, c.x, c.y, c.path))
        end
        if bestObj then
            qlog(string_format("  PICK: text=%q pos=(%d,%d) score=%d",
                (safeProp(bestObj, "Text") or ""):sub(1, 40),
                bestX or 0, bestY or 0, bestScore))
        end
    end

    if bestScore <= -400 then return nil, nil, nil end
    return bestX, bestY, bestObj
end

-- Find the Confirm button using STRICT exact-text matching.
-- Returns x, y, obj — or nil, nil, nil if nothing found.
findQuestConfirmButtonStrict = function()
    local bestObj, bestX, bestY, bestScore = nil, nil, nil, -99999
    local candidates = {}
    local roots = getQuestGuiRoots()

    for _, root in ipairs(roots) do
        for _, obj in ipairs(safeDescendants(root)) do
            local text = safeProp(obj, "Text")
            if type(text) == "string" and #text > 0 and questConfirmTextMatches(text) then
                local x, y = getGuiCenter(obj)
                if x and y then
                    local score = scoreQuestGuiCandidate(obj, x, y)
                    -- Bump score for exact text match
                    score = score + 200
                    table.insert(candidates, {
                        text = text:sub(1, 40), x = x, y = y, score = score,
                        path = safeFullName(obj):sub(1, 80),
                    })
                    if score > bestScore then
                        bestObj, bestX, bestY, bestScore = obj, x, y, score
                    end
                end
            end
        end
    end

    if #candidates > 0 then
        qlog(string_format("findQuestConfirmButtonStrict found %d candidates:", #candidates))
        for _, c in ipairs(candidates) do
            qlog(string_format("  score=%d text=%q pos=(%d,%d) path=%s",
                c.score, c.text, c.x, c.y, c.path))
        end
    end

    if bestScore <= 0 then return nil, nil, nil end
    return bestX, bestY, bestObj
end

-- Find the confirm button by scanning for any visible TextButton whose text
-- is one of the known confirm labels. Uses STRICT exact-text matching.
findQuestConfirmButton = function()
    return findQuestConfirmButtonStrict()
end

-- Polling helpers (used inside acceptQuestUi so we can time out gracefully)
function pollForGuiButton(wantedText, timeoutSeconds, intervalSeconds)
    local deadline = os_clock() + timeoutSeconds
    while os_clock() < deadline do
        local x, y, obj = findQuestGuiButton(wantedText)
        if x and y then return x, y, obj end
        task_wait(intervalSeconds or 0.05)
    end
    return nil, nil, nil
end

function pollForConfirmButton(timeoutSeconds, intervalSeconds)
    local deadline = os_clock() + timeoutSeconds
    while os_clock() < deadline do
        local x, y, obj = findQuestConfirmButtonStrict()
        if x and y then return x, y, obj end
        task_wait(intervalSeconds or 0.05)
    end
    return nil, nil, nil
end

-- ============================================================================
-- NEW DIALOG-LOCKED SCANNER (v2)
-- ============================================================================
-- The old scanner searched all of PlayerGui which picked up background UI
-- (Stats/Items/Shop menus, "QUEST" title labels, etc).
--
-- The new scanner:
--   1. Finds the DIALOG CONTAINER first (a Frame whose descendant text
--      contains "please select a quest" OR the documented Main.Dialogue)
--   2. Gets ALL TextButtons inside that container
--   3. Filters out cancel/close/nevermind/stats/items/shop buttons
--   4. Sorts remaining buttons by Y position (top to bottom)
--   5. Clicks the Nth button (option index, 1-based)
--
-- This eliminates ALL text-matching ambiguity. "Gorillas" vs "Gorilla" vs
-- "Gorilla King" doesn't matter — we just click "the 2nd button from the top".

-- Patterns that identify the dialog container (case-insensitive substring)
local DIALOG_MARKER_PATTERNS = {
    "please select a quest",
    "select a quest",
    "please select",
}

-- Patterns that disqualify a button from being a quest option button.
-- CRITICAL: Do NOT include "confirm", "accept", "yes", "start", "begin" here —
-- those are VALID buttons we want to click in Step D (after the option is selected,
-- the dialog switches to Confirm/Return state and we need to find the Confirm button).
-- This list is for actual CANCEL/close/navigation buttons only.
local NON_OPTION_BUTTON_PATTERNS = {
    "return", "nevermind", "never mind", "cancel", "close", "exit",
    "leave", "back", "stats", "items", "shop", "menu", "settings",
    "inventory", "help", "ok", "no",
    "buy", "sell", "trade", "talk",
    -- Title text fragments (from "Please select a quest." dialog title)
    "please", "select", "quest", "adventurer", "villager", "mayor",
}

-- (Forward declarations for these are at the top of the file.)

-- CACHE: once we find the dialog container, store it here so we don't re-scan
-- all of PlayerGui on every poll. Invalidated when the dialog closes.
local _cachedDialogContainer = nil
local _cachedDialogCheckedAt = 0
local _cachedDialogTTL = 2.0  -- re-validate cache after 2 seconds

-- Find the dialog container: the Frame that holds the "Please select a quest."
-- text and the Option1/Option2/Option3 buttons. In Blox Fruits this is
-- `PlayerGui.Main.Dialogue`.
--
-- CRITICAL: We do NOT check `Visible == true` because Matcha returns `nil`
-- (not `true`) for the Visible property of most GUI objects. We only reject
-- objects whose Visible is explicitly `false`.
--
-- Returns: the container Instance, or nil if not found.
findDialogContainer = function(opts)
    opts = opts or {}
    local forceRefresh = opts.forceRefresh == true
    local now = os_clock()

    -- Use cache if recent enough
    if not forceRefresh and _cachedDialogContainer then
        if (now - _cachedDialogCheckedAt) < _cachedDialogTTL then
            local stillValid = true
            pcall(function()
                local _ = _cachedDialogContainer.Name
            end)
            if stillValid then
                return _cachedDialogContainer
            end
            _cachedDialogContainer = nil
        end
    end

    local lp = getLocalPlayer()
    if not lp then return nil end
    local pg = safeFind(lp, "PlayerGui")
    if not pg then return nil end

    -- CRITICAL: In Blox Fruits, Main.Dialogue is an ImageLabel (not a Frame!),
    -- so we don't restrict by ClassName. We just look for it by name.
    --
    -- Confirmed structure (from MH2c dump):
    --   PlayerGui.Main.Dialogue [ImageLabel]
    --     ├── Option1 [TextButton]  ← quest option 1
    --     ├── Option2 [TextButton]  ← quest option 2
    --     ├── Option3 [TextButton]  ← quest option 3
    --     ├── Cancel  [TextButton]  ← "Nevermind"
    --     └── TextFrame              ← title text

    local main = safeFind(pg, "Main")
    if main then
        for _, name in ipairs({"Dialogue", "Dialog", "QuestDialog", "QuestDialogue", "NPCDialog"}) do
            local d = safeFind(main, name)
            if d then
                if safeProp(d, "Visible") ~= false then
                    -- Verify it has descendants with text content
                    for _, desc in ipairs(safeDescendants(d)) do
                        local t = safeProp(desc, "Text")
                        if type(t) == "string" and #t > 0 then
                            _cachedDialogContainer = d
                            _cachedDialogCheckedAt = now
                            return d
                        end
                    end
                end
            end
        end
    end

    -- Slow path: scan all of PlayerGui for a Frame containing a marker text.
    local candidates = {}
    for _, root in ipairs(getQuestGuiRoots()) do
        for _, desc in ipairs(safeDescendants(root)) do
            local cn = string_lower(safeProp(desc, "ClassName") or "")
            -- Include ImageLabel in the search (Main.Dialogue is an ImageLabel!)
            if cn == "frame" or cn == "scrollingframe" or cn == "screengui" or cn == "imagelabel" then
                if safeProp(desc, "Visible") ~= false then
                    local markerCount = 0
                    for _, d2 in ipairs(safeDescendants(desc)) do
                        local t = safeProp(d2, "Text")
                        if type(t) == "string" and #t > 0 then
                            local lower = string_lower(t)
                            for _, pat in ipairs(DIALOG_MARKER_PATTERNS) do
                                if string_find(lower, pat, 1, true) then
                                    markerCount = markerCount + 1
                                    break
                                end
                            end
                        end
                    end
                    if markerCount > 0 then
                        local depth = 0
                        local cur = desc
                        while cur do
                            depth = depth + 1
                            cur = safeProp(cur, "Parent")
                            if not cur then break end
                            local pcn = string_lower(safeProp(cur, "ClassName") or "")
                            if pcn == "playergui" or pcn == "datamodel" then break end
                        end
                        table.insert(candidates, {container = desc, count = markerCount, depth = depth})
                    end
                end
            end
        end
    end

    if #candidates == 0 then return nil end

    table.sort(candidates, function(a, b)
        if a.count ~= b.count then return a.count > b.count end
        return a.depth > b.depth
    end)
    _cachedDialogContainer = candidates[1].container
    _cachedDialogCheckedAt = now
    return _cachedDialogContainer
end

-- Invalidate the cached dialog container (call when dialog closes or after accept)
invalidateDialogCache = function()
    _cachedDialogContainer = nil
    _cachedDialogCheckedAt = 0
end

-- Check if a button's text matches any of the non-option patterns.
-- Returns true if the button should be EXCLUDED from option button list.
isNonOptionButton = function(text)
    if type(text) ~= "string" then return true end
    local lower = string_lower(text)
    for _, pat in ipairs(NON_OPTION_BUTTON_PATTERNS) do
        if string_find(lower, pat, 1, true) then
            return true
        end
    end
    return false
end

-- Get all "option-like" buttons inside a dialog container.
-- STRATEGY (v3 — based on actual Blox Fruits dialog structure observed):
--   Blox Fruits dialog buttons are named "Option1", "Option2", "Option3", etc.
--   The button itself has empty Text — the visible text ("Monkeys", "Gorillas",
--   "Gorilla King") lives on a CHILD TextLabel named "TextLabel".
--
--   So we look for descendants whose Name matches "^Option%d+$" and use those
--   directly. We also fall back to scanning TextLabels if no Option* children
--   are found (in case the dialog uses a different naming convention).
--
-- Returns a list sorted by Y position (top to bottom):
--   { {obj=..., x=..., y=..., text=..., name=...}, ... }
getOptionButtonsInDialog = function(dialogContainer)
    if not dialogContainer then return {} end

    local buttons = {}
    local unpositionedButtons = {}  -- buttons found by name but without position

    -- ─── PRIMARY PATH: find children named Option1, Option2, Option3, ... ───
    -- These are the actual clickable buttons in Blox Fruits.
    -- The button itself (Option1) has empty Text — visible text ("Monkeys")
    -- lives on its child TextLabel.
    --
    -- CRITICAL: The button's own AbsolutePosition may be nil in Matcha.
    -- We collect ALL Option buttons by name (even without positions), then
    -- estimate positions for the ones that are missing using the spacing
    -- of the ones that DID render.
    for _, desc in ipairs(safeDescendants(dialogContainer)) do
        local name = safeProp(desc, "Name") or ""
        -- Match "Option1" through "Option99" (covers all realistic quest givers)
        local optIdx = string.match(name, "^Option(%d+)$")
        if optIdx then
            optIdx = tonumber(optIdx)

            -- Try the button's own center first
            local cx, cy = getGuiCenter(desc)

            -- Try to get the visible text from child TextLabel
            local visibleText = ""
            local childLabel = safeFind(desc, "TextLabel")
            if childLabel then
                visibleText = safeProp(childLabel, "Text") or ""
                -- If button's own center failed, use the TextLabel's center
                if not (cx and cy) then
                    cx, cy = getGuiCenter(childLabel)
                end
            end

            if cx and cy then
                -- Walk upward to verify no BillboardGui/SurfaceGui in ancestry
                local hasWorldSpaceAncestor = false
                local current = desc
                local depth = 0
                while current and depth < 30 do
                    local acn = string_lower(safeProp(current, "ClassName") or "")
                    if acn == "billboardgui" or acn == "surfacegui" then
                        hasWorldSpaceAncestor = true
                        break
                    end
                    if safeProp(current, "Visible") == false then
                        hasWorldSpaceAncestor = true
                        break
                    end
                    local parent = safeProp(current, "Parent")
                    if not parent then break end
                    current = parent
                    depth = depth + 1
                end
                if not hasWorldSpaceAncestor then
                    table.insert(buttons, {
                        obj = desc,
                        x = cx,
                        y = cy,
                        text = visibleText,
                        name = name,
                        optIdx = optIdx,
                        path = safeFullName(desc):sub(1, 100),
                    })
                end
            else
                -- Button found by name but no position — save for position estimation
                table.insert(unpositionedButtons, {
                    obj = desc,
                    text = visibleText,
                    name = name,
                    optIdx = optIdx,
                })
            end
        end
    end

    -- ─── POSITION ESTIMATION for unpositioned buttons ───────────────────
    -- If we have at least 2 positioned buttons, we can estimate the Y spacing
    -- between option buttons and interpolate the position of missing ones.
    -- All option buttons share the same X position (they're in a vertical list).
    if #unpositionedButtons > 0 and #buttons >= 1 then
        -- Sort positioned buttons by optIdx
        table.sort(buttons, function(a, b) return a.optIdx < b.optIdx end)

        -- Calculate Y spacing from positioned buttons
        local ySpacing = 65  -- default spacing (observed: ~64-65 pixels)
        if #buttons >= 2 then
            -- Use the average spacing between consecutive positioned buttons
            local totalSpacing = 0
            local spacingCount = 0
            for i = 2, #buttons do
                local dy = buttons[i].y - buttons[i-1].y
                local dIdx = buttons[i].optIdx - buttons[i-1].optIdx
                if dIdx > 0 and dy > 0 then
                    totalSpacing = totalSpacing + (dy / dIdx)
                    spacingCount = spacingCount + 1
                end
            end
            if spacingCount > 0 then
                ySpacing = totalSpacing / spacingCount
            end
        end

        -- Use the first positioned button as reference
        local refBtn = buttons[1]
        local refX = refBtn.x
        local refY = refBtn.y
        local refIdx = refBtn.optIdx

        -- Estimate positions for unpositioned buttons
        for _, ub in ipairs(unpositionedButtons) do
            local estimatedY = refY + (ub.optIdx - refIdx) * ySpacing
            qlog(string_format("getOptionButtonsInDialog: estimating position for %s (optIdx=%d, text=%q) → (%d, %d) [spacing=%.1f]",
                ub.name, ub.optIdx, ub.text:sub(1, 30), refX, math_floor(estimatedY), ySpacing))
            table.insert(buttons, {
                obj = ub.obj,
                x = refX,
                y = math_floor(estimatedY),
                text = ub.text,
                name = ub.name,
                optIdx = ub.optIdx,
                path = "(estimated)",
            })
        end

        -- Re-sort by optIdx
        table.sort(buttons, function(a, b) return a.optIdx < b.optIdx end)
    end

    -- If we found Option* children, sort by their numeric index (Option1 < Option2 < Option3)
    if #buttons > 0 then
        table.sort(buttons, function(a, b) return a.optIdx < b.optIdx end)
        return buttons
    end

    -- ─── FALLBACK PATH: scan TextLabels with non-option text ─────────────
    -- For dialogs that don't use the "Option1/2/3" naming convention.
    -- We allow TextLabels here because the click will propagate to the parent
    -- button via Roblox's input system.
    --
    -- CRITICAL: Filter out title text fragments. In Blox Fruits, the title
    -- "Please select a quest." is split into individual character TextLabels
    -- by Matcha's text renderer (Group001="Please", Group003="select",
    -- Group007="quest.", etc). These are NOT option buttons.
    -- We exclude:
    --   - Any text matching the dialog title patterns ("please", "select", "quest")
    --   - Any name starting with "Group" or "Char" (the title text fragments)
    --   - Any text < 3 chars (too short to be a quest option like "Monkeys")
    local TITLE_FRAGMENTS = {"please", "select", "quest", "adventurer", "villager", "mayor", "giver"}
    for _, desc in ipairs(safeDescendants(dialogContainer)) do
        local cn = string_lower(safeProp(desc, "ClassName") or "")
        if cn == "textlabel" or cn == "textbutton" then
            local text = safeProp(desc, "Text")
            if type(text) == "string" and #text > 0 then
                local name = safeProp(desc, "Name") or ""
                -- Filter out cancel/close/nevermind/title patterns
                if not isNonOptionButton(text) then
                    -- Filter out title text fragments (Group001, Char001, etc)
                    local isTitleFragment = false
                    if string.find(name, "^Group%d+") or string.find(name, "^Char%d+") or string.find(name, "^Line%d+") then
                        isTitleFragment = true
                    end
                    -- Also check if text matches a title fragment word
                    if not isTitleFragment then
                        local textLower = string_lower(text)
                        for _, frag in ipairs(TITLE_FRAGMENTS) do
                            if textLower == frag or textLower == frag .. "." then
                                isTitleFragment = true
                                break
                            end
                        end
                    end
                    -- Filter out short text (real option buttons say "Monkeys", "Gorillas", etc)
                    if not isTitleFragment and #text >= 3 then
                        local cx, cy = getGuiCenter(desc)
                        if cx and cy then
                            if cx > 0 and cy > 0 then
                                table.insert(buttons, {
                                    obj = desc,
                                    x = cx,
                                    y = cy,
                                    text = text,
                                    name = name,
                                    optIdx = 0,
                                    path = safeFullName(desc):sub(1, 100),
                                })
                            end
                        end
                    end
                end
            end
        end
    end

    -- Sort fallback by Y position (top to bottom)
    table.sort(buttons, function(a, b) return a.y < b.y end)
    return buttons
end

-- Click the Nth option button (1-indexed) inside the dialog.
-- Returns true on success, false on failure.
-- CRITICAL: When the dialog just opened, AbsolutePosition may be nil for the
-- Option buttons (Matcha quirk). We retry up to 10 times with 200ms between
-- retries to give the dialog time to render.
-- CRITICAL 2: We don't just wait for ANY buttons — we wait for a button that
-- MATCHES our target (by optIdx or by enemy name text). If Option1 (Pirate)
-- hasn't rendered yet but Option2 (Brute) has, we keep waiting rather than
-- falling back to Brute.
clickOptionButtonByIndex = function(idx)
    local dialog = findDialogContainer()
    if not dialog then
        qlog("clickOptionButtonByIndex: no dialog container found")
        return false
    end
    qlog("clickOptionButtonByIndex: dialog container = " .. safeFullName(dialog):sub(1, 120))

    local buttons = {}
    local maxRetries = 10
    local retryDelay = 0.20
    local pick = nil

    for attempt = 1, maxRetries do
        invalidateDialogCache()
        dialog = findDialogContainer({ forceRefresh = true })
        if not dialog then
            qlog(string_format("clickOptionButtonByIndex: attempt %d — dialog disappeared", attempt))
            task_wait(retryDelay)
        else
            buttons = getOptionButtonsInDialog(dialog)

            -- Check if we have a MATCH (not just any buttons)
            -- Strategy 1: Match by optIdx
            for _, b in ipairs(buttons) do
                if b.optIdx == idx then
                    pick = b
                    break
                end
            end
            -- Strategy 2: Match by enemy name text
            if not pick and LevelState.RouteEnemy and #LevelState.RouteEnemy > 0 then
                for _, b in ipairs(buttons) do
                    if questOptionTextMatches(b.text, LevelState.RouteEnemy) then
                        pick = b
                        break
                    end
                end
            end

            if pick then
                -- Found our target button!
                if attempt > 1 then
                    qlog(string_format("clickOptionButtonByIndex: matched target on attempt %d (%d buttons visible)", attempt, #buttons))
                end
                break
            end

            -- We have buttons but none match our target. Keep retrying.
            if #buttons > 0 then
                qlog(string_format("clickOptionButtonByIndex: attempt %d — found %d buttons but none match target (optIdx=%d, enemy=%s), retrying",
                    attempt, #buttons, idx, tostring(LevelState.RouteEnemy)))
            else
                qlog(string_format("clickOptionButtonByIndex: attempt %d — found 0 buttons with positions, retrying", attempt))
            end
            task_wait(retryDelay)
        end
    end

    -- Log all buttons found
    qlog(string_format("clickOptionButtonByIndex: found %d option buttons in dialog:", #buttons))
    for i, b in ipairs(buttons) do
        qlog(string_format("  [%d] name=%q text=%q pos=(%d,%d) optIdx=%d",
            i, b.name, b.text:sub(1, 40), b.x, b.y, b.optIdx or 0))
    end

    -- If we still don't have a match after all retries, DON'T fall back to
    -- a wrong button. Abort and let the caller retry the whole accept flow.
    -- This is better than accepting the wrong quest (Brute instead of Pirate).
    if not pick then
        qlog(string_format("clickOptionButtonByIndex: target button (optIdx=%d, enemy=%s) never rendered after %d retries — aborting",
            idx, tostring(LevelState.RouteEnemy), maxRetries))
        return false
    end

    -- BOSS CHECK: If the picked button is a boss, try to find a non-boss alternative
    if isBossEnemy(pick.text) then
        qlog(string_format("clickOptionButtonByIndex: %q is a BOSS — looking for non-boss alternative",
            pick.text:sub(1, 40)))
        for i, b in ipairs(buttons) do
            if not isBossEnemy(b.text) then
                pick = b
                qlog(string_format("clickOptionButtonByIndex: switched to non-boss → [%d] %q",
                    i, b.text:sub(1, 40)))
                break
            end
        end
        if isBossEnemy(pick.text) then
            qlog("clickOptionButtonByIndex: ALL option buttons are bosses — aborting")
            return false
        end
    end

    qlog(string_format("clickOptionButtonByIndex: PICKED name=%q text=%q at (%d, %d)",
        pick.name, pick.text:sub(1, 40), pick.x, pick.y))

    -- Use guiClickAt (hold-click pattern — reliable for GUI buttons)
    local ok = guiClickAt(pick.x, pick.y, { holdTime = 0.08, settleTime = Config.QuestUiDelay })
    qlog(string_format("clickOptionButtonByIndex: clicked (ok=%s)", tostring(ok)))
    return ok
end

-- Find the Confirm button inside the dialog container.
-- STRATEGY: Blox Fruits buttons have empty Text on the parent + visible text on
-- a child TextLabel. So we check BOTH the button itself AND any child TextLabel.
-- Returns x, y, obj — or nil, nil, nil.
findConfirmButtonInDialog = function()
    -- In Blox Fruits, after clicking a quest option (e.g. Gorillas), the quest
    -- is accepted DIRECTLY — no Confirm button appears inside Main.Dialogue.
    -- The quest tracker (Main.Quest) becomes visible instead.
    --
    -- However, if a Confirm dialog DOES appear (for things like buying items),
    -- it's at Main.Confirm.Bottom.Confirm — a SIBLING of Main.Dialogue, NOT
    -- a child. So we check both places.
    --
    -- Returns x, y, obj — or nil, nil, nil.

    -- Strategy A: Check Main.Confirm.Bottom.Confirm (Blox Fruits' actual
    -- confirm dialog path, discovered via MH2c dump).
    local lp = getLocalPlayer()
    if lp then
        local pg = safeFind(lp, "PlayerGui")
        if pg then
            local main = safeFind(pg, "Main")
            if main then
                local confirmFrame = safeFind(main, "Confirm")
                if confirmFrame then
                    -- Check if Main.Confirm is visible
                    if safeProp(confirmFrame, "Visible") ~= false then
                        local bottom = safeFind(confirmFrame, "Bottom")
                        if bottom then
                            local confirmBtn = safeFind(bottom, "Confirm")
                            if confirmBtn then
                                local cx, cy = getGuiCenter(confirmBtn)
                                if cx and cy then
                                    qlog(string_format("findConfirmButtonInDialog: FOUND Main.Confirm.Bottom.Confirm at (%d, %d)", cx, cy))
                                    return cx, cy, confirmBtn
                                end
                            end
                        end
                    end
                end
            end
        end
    end

    -- Strategy B: Look for a child named "Confirm" or "Accept" inside the
    -- dialogue container (in case Blox Fruits changes the structure later).
    local dialog = findDialogContainer()
    if dialog then
        for _, name in ipairs({"Confirm", "Accept", "Yes", "Start", "BeginQuest", "Begin"}) do
            local btn = safeFind(dialog, name)
            if btn then
                local cx, cy = getGuiCenter(btn)
                if cx and cy then
                    qlog(string_format("findConfirmButtonInDialog: FOUND by name=%q at (%d, %d)", name, cx, cy))
                    return cx, cy, btn
                end
            end
        end
    end

    return nil, nil, nil
end

-- Polling version of findConfirmButtonInDialog
-- OPTIMIZED: uses cached dialog container, exits early if quest tracker appears,
-- logs only ONCE at the start (not on every poll) to reduce console spam.
pollForConfirmButtonInDialog = function(timeoutSeconds, intervalSeconds)
    local deadline = os_clock() + timeoutSeconds
    local pollInterval = intervalSeconds or 0.20  -- was 0.05 (20Hz); now 5Hz — way less laggy
    local loggedStart = false
    local pollCount = 0
    while os_clock() < deadline do
        pollCount = pollCount + 1
        -- Log only the first poll so we know scanning started
        if not loggedStart then
            qlog(string_format("pollForConfirmButtonInDialog: START (timeout=%.1fs, interval=%.2fs)",
                timeoutSeconds, pollInterval))
            loggedStart = true
        end
        local x, y, obj = findConfirmButtonInDialog()
        if x and y then
            qlog(string_format("pollForConfirmButtonInDialog: FOUND after %d polls", pollCount))
            return x, y, obj
        end
        -- EXIT EARLY: if quest tracker became visible, the option click already
        -- accepted the quest — no Confirm button needed. Stop polling.
        if isQuestTrackerVisible() then
            qlog(string_format("pollForConfirmButtonInDialog: quest tracker appeared after %d polls — stopping early", pollCount))
            return nil, nil, nil
        end
        -- EXIT EARLY: if dialog closed, the option click worked but tracker is pending
        if not isQuestDialogOpen() then
            qlog(string_format("pollForConfirmButtonInDialog: dialog closed after %d polls — stopping early", pollCount))
            return nil, nil, nil
        end
        task_wait(pollInterval)
    end
    qlog(string_format("pollForConfirmButtonInDialog: TIMEOUT after %d polls", pollCount))
    return nil, nil, nil
end

-- Diagnostic: dump every Text object the script can see in PlayerGui +
-- CoreGui + StarterGui. Used by the "Dump GUI text" UI button so we can
-- see exactly what Blox Fruits' dialog looks like to the script.
dumpAllGuiText = function()
    qlog("=== DUMPING ALL GUI TEXT OBJECTS ===")
    local roots = getQuestGuiRoots()
    local count = 0
    for _, root in ipairs(roots) do
        local rootName = safeProp(root, "Name") or "?"
        local rootClass = safeProp(root, "ClassName") or "?"
        qlog(string_format("ROOT: %s (class=%s)", tostring(rootName), tostring(rootClass)))
        for _, obj in ipairs(safeDescendants(root)) do
            local className = safeProp(obj, "ClassName") or ""
            local name = safeProp(obj, "Name") or ""
            local text = safeProp(obj, "Text")
            local visible = safeProp(obj, "Visible")
            -- only dump objects that have a Text property with string content
            if type(text) == "string" and #text > 0 then
                local x, y = getGuiCenter(obj)
                local fullName = safeFullName(obj)
                local valid = isValidQuestButton(obj)
                qlog(string_format("  [%s] name=%q text=%q visible=%s valid=%s pos=(%s,%s) path=%s",
                    className,
                    tostring(name):sub(1, 30),
                    text:sub(1, 80),
                    tostring(visible),
                    tostring(valid),
                    tostring(x), tostring(y),
                    fullName:sub(1, 140)
                ))
                count = count + 1
            end
        end
    end
    qlog(string_format("=== TOTAL TEXT OBJECTS DUMPED: %d ===", count))
    return count
end

-- Detect whether the quest dialog is currently open.
-- We use 3 independent signals and return true if ANY of them fire:
--   1. PlayerGui.Main.Dialogue.Visible == true (the documented path)
--   2. Any descendant of PlayerGui.Main has Text containing "please select a quest"
--   3. Any visible TextButton inside PlayerGui whose Text contains the target
--      enemy name AND is positioned in the right half of the screen
--      (the dialog option buttons are labeled with the enemy name)
local DIALOG_TITLE_PATTERNS = {
    "please select a quest",
    "select a quest",
    "please select",
}

isQuestDialogOpen = function()
    local lp = getLocalPlayer()
    if not lp then return false end
    local pg = safeFind(lp, "PlayerGui")
    if not pg then return false end

    -- Signal 1: PlayerGui.Main.Dialogue.Visible
    -- (Main.Dialogue is an ImageLabel in Blox Fruits, but Visible still works)
    -- CRITICAL: Matcha returns nil for Visible (not true), so we accept
    -- Visible ~= false as "visible".
    local main = safeFind(pg, "Main")
    if main then
        local dialogue = safeFind(main, "Dialogue")
        if dialogue and safeProp(dialogue, "Visible") ~= false then
            -- Verify it has descendants (so we don't false-positive on empty hidden ImageLabels)
            local hasContent = false
            for _, desc in ipairs(safeDescendants(dialogue)) do
                local t = safeProp(desc, "Text")
                if type(t) == "string" and #t > 0 then
                    hasContent = true
                    break
                end
                -- Also count TextButtons (Option1/2/3) as content
                local cn = string_lower(safeProp(desc, "ClassName") or "")
                if cn == "textbutton" then
                    hasContent = true
                    break
                end
            end
            if hasContent then return true end
        end
        -- Also try alternate spellings
        for _, altName in ipairs({"Dialog", "QuestDialog", "QuestDialogue", "NPCDialog"}) do
            local alt = safeFind(main, altName)
            if alt and safeProp(alt, "Visible") ~= false then
                local hasText = false
                for _, desc in ipairs(safeDescendants(alt)) do
                    local t = safeProp(desc, "Text")
                    if type(t) == "string" and #t > 0 then
                        hasText = true
                        break
                    end
                end
                if hasText then return true end
            end
        end
    end

    -- Signal 2: scan Main descendants for the "Please select a quest" title text
    if main then
        for _, desc in ipairs(safeDescendants(main)) do
            if safeProp(desc, "Visible") ~= false then
                local t = safeProp(desc, "Text")
                if type(t) == "string" and #t > 0 then
                    local lower = string_lower(t)
                    for _, pat in ipairs(DIALOG_TITLE_PATTERNS) do
                        if string_find(lower, pat, 1, true) then
                            return true
                        end
                    end
                end
            end
        end
    end

    -- Signal 3: any visible TextButton with target enemy text in right half of screen
    if LevelState.RouteEnemy and type(LevelState.RouteEnemy) == "string" and #LevelState.RouteEnemy > 0 then
        local targetLower = string_lower(LevelState.RouteEnemy)
        local vpX = getViewportSize()
        if main then
            for _, desc in ipairs(safeDescendants(main)) do
                local cn = string_lower(safeProp(desc, "ClassName") or "")
                if cn == "textbutton" and safeProp(desc, "Visible") ~= false then
                    local t = safeProp(desc, "Text")
                    if type(t) == "string" and #t > 0 then
                        local lower = string_lower(t)
                        if string_find(lower, targetLower, 1, true) then
                            local cx, cy = getGuiCenter(desc)
                            if cx and cy and cx > vpX * 0.40 then
                                return true
                            end
                        end
                    end
                end
            end
        end
    end

    return false
end

-- Detect whether the active quest tracker is visible
-- CRITICAL: We check the SPECIFIC quest title text at:
--   PlayerGui.Main.Quest.Container.QuestTitle.Title
-- If that TextLabel is missing OR has empty text, the quest is complete.
-- We DON'T just check for ANY text in the Quest container because the
-- QuestReward frame and other UI elements may have text even when the
-- quest title is gone.
isQuestTrackerVisible = function()
    local title = getCachedQuestTitle()
    if not title then return false end
    local quest = getCachedQuestFrame()
    if quest and safeProp(quest, "Visible") == false then return false end

    local titleText = safeProp(title, "Text")

    -- CRITICAL: Matcha returns "failed to fetch text" when a TextLabel is
    -- cleared or hidden. We must treat this as "no text" = quest complete.
    -- Also reject empty string, nil, and any text shorter than 5 chars
    -- (a real quest title is always "Defeat X Y (Z/W)" which is 15+ chars).
    if type(titleText) ~= "string" then return false end
    if #titleText < 5 then return false end
    if string_find(string_lower(titleText), "failed to fetch", 1, true) then return false end
    if string_find(string_lower(titleText), "nil", 1, true) and #titleText < 10 then return false end

    return true
end

-- Read the quest tracker title text (contains the kill counter).
-- e.g. "Defeat 8 Chief Petty Officers (0/8)" — return that string or nil.
getQuestTrackerText = function()
    local title = getCachedQuestTitle()
    if not title then return nil end
    local text = safeProp(title, "Text")
    -- CRITICAL: Matcha returns "failed to fetch text" when a TextLabel is cleared.
    -- Treat this as nil (no text = quest complete).
    if type(text) ~= "string" then return nil end
    if #text < 5 then return nil end
    if string_find(string_lower(text), "failed to fetch", 1, true) then return nil end
    return text
end

-- Returns true when the quest tracker text contains the expected enemy name
questTrackerMatchesEnemy = function()
    local text = getQuestTrackerText()
    if type(text) ~= "string" or #text < 1 then return false end
    return string_find(string_lower(text), string_lower(LevelState.RouteEnemy or ""), 1, true) ~= nil
end

-- ============================================================================
-- 17. QUEST ACCEPTANCE ( Step 2 of the 3-step flow )
-- ============================================================================

acceptQuestUi = function()
    qlog("acceptQuestUi START | enemy=" .. tostring(LevelState.RouteEnemy) ..
         " option=" .. tostring(LevelState.RouteOption) ..
         " giver=" .. tostring(LevelState.RouteQuestGiver))

    -- Wait for the dialog to actually be open (we should have just clicked the giver)
    qlog("Waiting for dialog to open: " .. tostring(Config.QuestUiDelay * 3) .. "s")
    local openedAt = os_clock()
    while os_clock() - openedAt < (Config.QuestUiDelay * 30) do
        if isQuestDialogOpen() then break end
        task_wait(0.05)
    end

    if not isQuestDialogOpen() then
        qlog("Dialog did NOT open after clicking giver — aborting accept")
        setStatus("Quest dialog did not open")
        return false
    end
    qlog("Dialog is open — proceeding to option scan (v2 dialog-locked scanner)")

    -- ════════════════════════════════════════════════════════════════════
    -- STEP A: Click the quest option button by index (e.g. Option2 = Gorillas)
    -- ════════════════════════════════════════════════════════════════════
    setStatus(string_format("Quest: clicking option %d (of %s)",
        LevelState.RouteOption, tostring(LevelState.RouteEnemy)))
    qlog(string_format("Step A: clicking option button by index=%d (enemy=%s)",
        LevelState.RouteOption, tostring(LevelState.RouteEnemy)))

    local clicked = clickOptionButtonByIndex(LevelState.RouteOption)
    if not clicked then
        qlog("Step A FAILED: could not click option button by index")
        qlog("Step A FAILED: dumping ALL visible GUI text for diagnosis:")
        dumpAllGuiText()
        setStatus("Quest option click failed (dumped GUI)")
        return false
    end
    qlog("Step A SUCCESS: option button clicked")

    -- Step C: wait 1.70 seconds (your requested delay).
    -- In Blox Fruits, clicking an option (e.g. Gorillas) does NOT directly
    -- accept the quest — instead, the dialog CHANGES to show "Confirm" / "Return"
    -- buttons (reusing Option1/Option2). We need to click "Confirm" next.
    setStatus(string_format("Quest: waiting %.2fs for Confirm state", Config.QuestConfirmDelay))
    qlog(string_format("Step C: waiting %.2fs for dialog to switch to Confirm/Return state",
        Config.QuestConfirmDelay))
    task_wait(Config.QuestConfirmDelay)

    -- ════════════════════════════════════════════════════════════════════
    -- STEP D: Look for a "Confirm" button in the dialog and click it.
    -- After clicking Gorillas, the dialog reuses Option1/Option2 with new text:
    --   Option1 → "Confirm"  (we want to click this)
    --   Option2 → "Return"   (cancel)
    -- We scan by TEXT (not index) to find the Confirm button.
    -- ════════════════════════════════════════════════════════════════════
    setStatus("Quest: scanning for Confirm button")
    qlog("Step D: scanning for Confirm button by text in dialog option buttons")

    -- Invalidate cache because dialog contents changed (Option2 now says "Return")
    invalidateDialogCache()

    local confirmClicked = false
    -- CRITICAL: Blox Fruits' actual Confirm button text is "Close" (verified
    -- via MH2c dump: Main.Confirm.Bottom.Confirm.TextLabel text="Close").
    -- We accept any of these labels for the confirm action.
    local confirmLabels = {
        "Confirm", "Accept", "Yes", "Start", "Start Quest", "Begin Quest", "Begin",
        "Close", "OK", "Okay", "Done", "Continue",
    }

    -- Helper: check if Main.Confirm is ACTUALLY visible (not just present in the tree).
    -- Main.Confirm always exists in PlayerGui but is hidden when not in use.
    -- We detect "actually visible" by checking if Main.Confirm.Content.TextLabel
    -- has non-empty text (when the dialog is showing, it has a message like
    -- "Are you sure you want to accept this quest?").
    -- CRITICAL: We also check if Main.Dialogue is still visible — if it is,
    -- then Main.Confirm might be a leftover from a previous dialog (like an
    -- item unbox notification). We only treat Main.Confirm as the quest confirm
    -- if Main.Dialogue has CLOSED (the quest dialog closes when Main.Confirm
    -- appears for quest acceptance).
    local function isMainConfirmActuallyVisible()
        local lp = getLocalPlayer()
        if not lp then return false, nil, nil, "", false end
        local pg = safeFind(lp, "PlayerGui")
        if not pg then return false, nil, nil, "", false end
        local main = safeFind(pg, "Main")
        if not main then return false, nil, nil, "", false end
        local confirmFrame = safeFind(main, "Confirm")
        if not confirmFrame then return false, nil, nil, "", false end
        -- Check Visible property (Matcha returns nil; treat ~= false as visible)
        if safeProp(confirmFrame, "Visible") == false then return false, nil, nil, "", false end
        -- Check Content.TextLabel has non-empty text
        local content = safeFind(confirmFrame, "Content")
        if not content then return false, nil, nil, "", false end
        local contentLabel = safeFind(content, "TextLabel")
        if not contentLabel then return false, nil, nil, "", false end
        local contentText = safeProp(contentLabel, "Text")
        if type(contentText) ~= "string" or #contentText < 1 then
            return false, nil, nil, "", false
        end
        -- Check if Main.Dialogue is still visible. If it is, Main.Confirm might be
        -- a leftover popup (item unbox, etc) — we mark it as "low confidence".
        local dialogueStillOpen = false
        local dialogue = safeFind(main, "Dialogue")
        if dialogue then
            local dialogueVisible = safeProp(dialogue, "Visible")
            if dialogueVisible ~= false then
                -- Check if Dialogue still has option buttons (indicating it's the quest dialog)
                local opt1 = safeFind(dialogue, "Option1")
                if opt1 then
                    dialogueStillOpen = true
                end
            end
        end
        -- Main.Confirm is showing! Get the Confirm button position.
        local bottom = safeFind(confirmFrame, "Bottom")
        if not bottom then return true, nil, nil, "", dialogueStillOpen end
        local confirmBtn = safeFind(bottom, "Confirm")
        if not confirmBtn then return true, nil, nil, "", dialogueStillOpen end
        -- Try button's own center first
        local cx, cy = getGuiCenter(confirmBtn)
        -- Fall back to child TextLabel position
        if not (cx and cy) then
            local childLabel = safeFind(confirmBtn, "TextLabel")
            if childLabel then
                cx, cy = getGuiCenter(childLabel)
            end
        end
        -- Get button text for logging
        local btnText = ""
        local childLabel = safeFind(confirmBtn, "TextLabel")
        if childLabel then
            btnText = safeProp(childLabel, "Text") or ""
        end
        return true, cx, cy, btnText, contentText, dialogueStillOpen
    end

    -- Poll for up to Config.QuestConfirmScanTime seconds
    local deadline = os_clock() + Config.QuestConfirmScanTime
    local pollCount = 0
    while os_clock() < deadline do
        pollCount = pollCount + 1

        -- ─── Strategy 1: Check if Main.Confirm is actually visible ───
        -- After clicking Gorillas, Main.Confirm appears ON TOP of Main.Dialogue
        -- with a Confirm button at Main.Confirm.Bottom.Confirm.
        -- CRITICAL: Only click Main.Confirm if Main.Dialogue has CLOSED.
        -- If Main.Dialogue is still open (with Option1/2/3 visible), then
        -- Main.Confirm might be a leftover popup (item unbox, etc) — don't click it.
        local visible, cx, cy, btnText, contentText, dialogueStillOpen = isMainConfirmActuallyVisible()
        if visible and cx and cy then
            if dialogueStillOpen then
                -- Main.Confirm is visible BUT Main.Dialogue is still open.
                -- This is likely a leftover popup, NOT the quest confirm.
                -- Only log on first poll to avoid spam.
                if pollCount == 1 then
                    qlog(string_format("Step D poll %d: Main.Confirm VISIBLE but Main.Dialogue still open (content=%q) — waiting for Dialogue to close",
                        pollCount, contentText:sub(1, 50)))
                end
            else
                -- Main.Confirm is visible AND Main.Dialogue has closed.
                -- This IS the quest confirm dialog.
                qlog(string_format("Step D poll %d: Main.Confirm VISIBLE (content=%q) — clicking Confirm button (text=%q) at (%d, %d)",
                    pollCount, contentText:sub(1, 50), btnText:sub(1, 40), cx, cy))
                local ok = guiClickAt(cx, cy, { holdTime = 0.08, settleTime = Config.QuestUiDelay })
                qlog(string_format("Step D: clicked Confirm (ok=%s)", tostring(ok)))
                confirmClicked = true
                break
            end
        end

        -- ─── Strategy 2: Scan option buttons in dialog for confirm-label text ───
        -- After clicking Gorillas, the dialog may switch to Confirm/Return state
        -- where Option1 text="Confirm" and Option2 text="Return".
        local dialog = findDialogContainer({ forceRefresh = true })
        if dialog then
            local buttons = getOptionButtonsInDialog(dialog)
            if #buttons > 0 then
                qlog(string_format("Step D poll %d: found %d option buttons in dialog", pollCount, #buttons))
                for i, b in ipairs(buttons) do
                    qlog(string_format("  [%d] name=%q text=%q pos=(%d,%d)",
                        i, b.name, b.text:sub(1, 40), b.x, b.y))
                    -- Check if this button's text matches a confirm label
                    local textLower = string_lower(b.text)
                    for _, label in ipairs(confirmLabels) do
                        if textLower == string_lower(label) then
                            qlog(string_format("Step D: FOUND Confirm button [%d] %q at (%d, %d) — clicking",
                                i, b.text:sub(1, 40), b.x, b.y))
                            local ok = guiClickAt(b.x, b.y, { holdTime = 0.08, settleTime = Config.QuestUiDelay })
                            qlog(string_format("Step D: clicked Confirm (ok=%s)", tostring(ok)))
                            confirmClicked = true
                            break
                        end
                    end
                    if confirmClicked then break end
                end
            end
        end
        if confirmClicked then break end

        -- ─── Strategy 3: Check if quest tracker appeared with correct enemy ───
        if isQuestTrackerVisible() and questTrackerMatchesEnemy() then
            qlog("Step D: quest tracker appeared with correct enemy — option directly accepted")
            confirmClicked = true  -- treat as confirmed
            break
        end

        -- ─── Strategy 4: Check if dialog closed (option worked, tracker pending) ───
        if not isQuestDialogOpen() then
            qlog("Step D: dialog closed — option may have directly accepted")
            break
        end

        task_wait(0.20)
    end

    if not confirmClicked then
        qlog("Step D: no Confirm button found in dialog. Dumping all visible GUI text:")
        dumpAllGuiText()
        setStatus("Quest: Confirm button not found (dumped GUI to console)")
        return false
    end

    -- ════════════════════════════════════════════════════════════════════
    -- STEP E: Verify quest tracker is visible AND matches the expected enemy.
    -- CRITICAL: isQuestTrackerVisible() can return true if an OLD quest is
    -- still active. We must ALSO check questTrackerMatchesEnemy() to ensure
    -- the new quest (Gorillas) was actually accepted.
    -- ════════════════════════════════════════════════════════════════════
    setStatus("Quest: verifying quest tracker")
    qlog("Step E: waiting for quest tracker to show enemy=" .. tostring(LevelState.RouteEnemy))
    task_wait(Config.QuestUiDelay * 2)

    local verifyDeadline = os_clock() + 5.0
    local trackerVisible = false
    local trackerMatches = false
    while os_clock() < verifyDeadline do
        trackerVisible = isQuestTrackerVisible()
        trackerMatches = questTrackerMatchesEnemy()
        if trackerVisible and trackerMatches then
            break
        end
        task_wait(0.10)
    end

    qlog(string_format("Step E: trackerVisible=%s trackerMatches=%s",
        tostring(trackerVisible), tostring(trackerMatches)))

    if trackerVisible and trackerMatches then
        LevelState.QuestAccepted = true
        LevelState.LastQuestAt   = os_clock()
        LevelState.QuestPhase    = "fight"
        LevelState.EnemyApproachDone = false
        qlog("acceptQuestUi COMPLETE — quest tracker visible AND matches enemy")
        setStatus("Quest accepted: " .. tostring(LevelState.RouteEnemy))
        notifyUser("Quest accepted: " .. tostring(LevelState.RouteEnemy), 4)
        return true
    end

    -- Tracker visible but doesn't match — wrong quest was accepted.
    -- Need to abandon and retry.
    if trackerVisible and not trackerMatches then
        local trackerText = getQuestTrackerText() or "(no text)"
        qlog("acceptQuestUi PARTIAL — tracker visible but text doesn't match enemy")
        qlog("  Tracker text: " .. tostring(trackerText):sub(1, 100))
        qlog("  Expected enemy: " .. tostring(LevelState.RouteEnemy))
        setStatus("Quest: wrong quest accepted (tracker mismatch)")
        -- Don't return true — let the caller retry
        -- The questGiverStep will see QuestAccepted=false and re-attempt
        return false
    end

    qlog("acceptQuestUi FAILED — quest tracker did NOT become visible")
    setStatus("Quest acceptance verification failed")
    invalidateDialogCache()
    return false
end

-- ============================================================================
-- 18. LEVEL FARM — THE 3-STEP FLOW
-- ============================================================================

questNeedsRefresh = function(now)
    if Config.LevelMode ~= "Quest" then return false end
    if not Config.LevelAutoRoute then return false end
    if not Config.QuestGiverEnabled then return false end
    if not LevelState.QuestAccepted then return true end
    -- CRITICAL: Don't refresh if the quest is still in progress!
    -- Check the quest tracker — if it's visible AND matches our enemy,
    -- the quest is still active and we should NOT re-accept it.
    if isQuestTrackerVisible() and questTrackerMatchesEnemy() then
        return false
    end
    if Config.QuestRefreshTime <= 0 then return false end
    return now - LevelState.LastQuestAt >= Config.QuestRefreshTime
end

-- Returns true if the player is close to a position (within `range` studs)
isPlayerNear = function(pos, range)
    local root = getLocalRoot()
    if not root or not pos then return false end
    local rootPos = safeProp(root, "Position")
    if not rootPos then return false end
    return distSq(rootPos, pos) <= (range * range)
end

-- STEP 1: If quest giver is missing, tween to the island's approach waypoint
--         so the island streams in and the giver NPC loads.
ensureQuestGiverAvailable = function()
    local giver, giverPos = findQuestGiver()
    if giver and giverPos then
        LevelState.QuestGiverObject   = safeProp(giver, "Name") or getWantedQuestGiver()
        LevelState.QuestGiverPosition = giverPos
        return true, giver, giverPos
    end

    -- Giver not found — tween to the island's approach waypoint if we're not close
    local island = LevelState.RouteIsland
    local wp = IslandWaypoints[island]
    if not wp then
        qlog("No island waypoint for: " .. tostring(island))
        return false, nil, nil
    end

    if isPlayerNear(wp, 30) then
        -- We're at the waypoint but giver still isn't loaded — wait a bit
        qlog("At island waypoint but giver not yet loaded: " .. tostring(island))
        return false, nil, nil
    end

    if LevelState.TweenSignal and not LevelState.TweenSignal.Completed then
        if LevelState.TweenPurpose == "island" then return false, nil, nil end
    end

    local root = getLocalRoot()
    if not root then return false, nil, nil end
    qlog("Tweening to island waypoint: " .. tostring(island))
    LevelState.TweenSignal  = tweenRootTo(root, wp, wp, Config.EnemyApproachSpeed, Config.LevelTweenEasing)
    LevelState.TweenPurpose = "island"
    LevelState.QuestPhase   = "to_island"
    setStatus("Traveling to " .. tostring(island) .. " to load quest giver")
    return false, nil, nil
end

-- STEP 2: Tween to quest giver, click it, run acceptQuestUi()
questGiverStep = function(now)
    if not questNeedsRefresh(now) then return false end

    -- If a tween is in progress, wait for it
    if LevelState.TweenSignal and not LevelState.TweenSignal.Completed then
        if LevelState.TweenPurpose == "quest_giver" then
            LevelState.QuestPhase = "to_giver"
            setStatus("Quest: moving to " .. tostring(LevelState.RouteQuestGiver))
        end
        return true
    end

    -- If we just finished tweening to the giver, click it and run accept flow
    if LevelState.TweenPurpose == "quest_giver" and LevelState.TweenSignal and LevelState.TweenSignal.Completed then
        -- Cooldown check: don't re-attempt accept too quickly after a failure
        local nowSecs = os_clock()
        if LevelState.LastAcceptFailAt > 0 and (nowSecs - LevelState.LastAcceptFailAt) < LevelState.AcceptCooldownSecs then
            LevelState.QuestPhase = "cooldown"
            setStatus(string_format("Quest accept cooldown (%.1fs remaining)",
                LevelState.AcceptCooldownSecs - (nowSecs - LevelState.LastAcceptFailAt)))
            return true  -- wait
        end

        -- Auto-pause check: if too many consecutive fails, pause LevelProcess
        if LevelState.AcceptFailCount >= LevelState.AcceptMaxFails then
            LevelState.AutoPaused = true
            Config.LevelProcess = false
            LevelState.QuestPhase = "paused"
            qlog(string_format("AUTO-PAUSED after %d consecutive accept failures",
                LevelState.AcceptFailCount))
            notifyUser("Level farm auto-paused after " .. tostring(LevelState.AcceptFailCount) ..
                       " failed quest accepts. Check F9 console, then click 'Reset quest state' " ..
                       "and re-enable Level process.", 10)
            return true
        end

        LevelState.QuestPhase = "quest_ui"
        qlog("Arrived at giver — clicking giver NPC")
        if LevelState.QuestGiverPosition then
            local clicked = clickWorldPosition(LevelState.QuestGiverPosition)
            if not clicked then
                qlog("Giver click FAILED (WorldToScreen returned bogus coords) — aborting this attempt")
                LevelState.LastAcceptFailAt = os_clock()
                LevelState.AcceptFailCount   = LevelState.AcceptFailCount + 1
                LevelState.TweenPurpose      = "none"
                setStatus("Giver click missed — will retry after cooldown")
                return true
            end
            -- Wait for the dialog to actually appear AND render its buttons.
            -- Matcha's AbsolutePosition returns nil until the GUI has fully
            -- rendered (usually takes 200-400ms after the click).
            task_wait(0.1)
        end
        local acceptOk = acceptQuestUi()
        if not acceptOk then
            LevelState.LastAcceptFailAt = os_clock()
            LevelState.AcceptFailCount  = LevelState.AcceptFailCount + 1
            qlog(string_format("acceptQuestUi FAILED (fail count: %d/%d)",
                LevelState.AcceptFailCount, LevelState.AcceptMaxFails))
        else
            LevelState.AcceptFailCount = 0
        end
        LevelState.TweenPurpose = "none"
        return true
    end

    -- Make sure the giver is available (Step 1 logic)
    local ok, giver, giverPos = ensureQuestGiverAvailable()
    if not ok then
        -- Either tweening to island or giver still loading
        if LevelState.QuestPhase ~= "to_island" and LevelState.QuestPhase ~= "giver_missing" then
            LevelState.QuestPhase = "giver_missing"
            setStatus("Quest giver missing: " .. tostring(getWantedQuestGiver()))
        end
        return true
    end

    -- If we're already at the giver (within 8 studs), DON'T re-tween — go
    -- straight to the click+accept branch on the next call. This prevents the
    -- "instant re-tween → instant arrive → instant fail → instant re-tween" loop.
    if isPlayerNear(giverPos, 8) then
        qlog("Already at giver (within 8 studs) — skipping tween, going straight to accept")
        -- synthesize a completed tween signal so the next call hits the accept branch
        LevelState.TweenSignal  = { Completed = true }
        LevelState.TweenPurpose = "quest_giver"
        LevelState.QuestPhase   = "quest_ui"
        return true
    end

    -- We have the giver — tween to it
    local giverName = safeProp(giver, "Name") or getWantedQuestGiver()
    LevelState.QuestGiverObject   = giverName
    LevelState.QuestGiverPosition = giverPos
    qlog("Found giver: " .. tostring(giverName) .. " at " .. tostring(giverPos))

    local root, standPos, lookPos = getStandPositionNear(
        giverPos, Config.QuestGiverDistance, Config.QuestGiverHeightOffset)
    if not root or not standPos or not lookPos then
        LevelState.QuestPhase = "no_root"
        setStatus("Quest: no root")
        return false
    end

    LevelState.TweenSignal  = tweenRootTo(root, standPos, lookPos)
    LevelState.TweenPurpose = "quest_giver"
    LevelState.QuestPhase   = "to_giver"
    setStatus("Quest: " .. tostring(giverName))
    return true
end

-- STEP 3: Float above enemy spawn, scan for mob, chase and kill
fightStep = function()
    -- If quest not accepted, do nothing (questGiverStep handles it)
    if not LevelState.QuestAccepted then return end

    -- Read the quest tracker text and parse the kill counter.
    local trackerText = getQuestTrackerText() or ""
    local killCurrent, killTotal = string_match(tostring(trackerText), "%((%d+)%s*/%s*(%d+)%)")
    if killCurrent and killTotal then
        killCurrent = tonumber(killCurrent) or 0
        killTotal = tonumber(killTotal) or 0

        -- Capture previous kill count BEFORE updating (for light rescan trigger).
        -- This is necessary because the update block below overwrites
        -- LevelState._lastKillCount with the new value, which would make the
        -- "killCurrent > _lastKillCount" check always false.
        local previousKillCount = LevelState._lastKillCount

        -- Track kill count changes for stuck detection
        if LevelState._lastKillCount ~= killCurrent then
            LevelState._lastKillCount = killCurrent
            LevelState._lastKillChangeAt = os_clock()
        end

        -- LIGHT RESCAN ON KILL: When the kill count goes up by +1 (or more),
        -- do a fast scan of ONLY Workspace.Enemies[RouteEnemy] to find the
        -- next target immediately. This is much cheaper than the full
        -- scanNPCs() (which iterates ALL enemies + NPCs in range).
        --
        -- SKIP the last kill (killCurrent == killTotal) because at that point
        -- the quest is complete and the tracker is about to disappear —
        -- there's nothing left to scan for. (As the user noted, you can't
        -- "see" the last kill happen because the tracker vanishes.)
        if Config.SmartRescan
           and previousKillCount ~= nil
           and killCurrent > previousKillCount
           and killTotal > 0
           and killCurrent < killTotal then
            local delta = killCurrent - previousKillCount
            local n = scanTargetEnemyFolder()
            qlog(string_format(
                "Light rescan: +%d kill(s) %d -> %d, found %d %s in target folder",
                delta, previousKillCount, killCurrent, n,
                tostring(LevelState.RouteEnemy)))
        end

        -- Log kill progress every 5 seconds (rate-limited)
        if not LevelState._lastKillLogAt or (os_clock() - LevelState._lastKillLogAt) > 5.0 then
            qlog(string_format("Quest progress: %d/%d %s (%s)",
                killCurrent, killTotal, tostring(LevelState.RouteEnemy),
                tostring(trackerText):sub(1, 60)))
            LevelState._lastKillLogAt = os_clock()
        end

        -- STUCK DETECTION: If kill count hasn't changed for 15 seconds.
        if LevelState._lastKillChangeAt and (os_clock() - LevelState._lastKillChangeAt) > 15.0 then
            if killCurrent < killTotal then
                -- If we're at killTotal-1 (e.g. 7/8), the last mob was probably
                -- killed but the tracker didn't update. Just mark the quest as
                -- complete — 30 seconds of no progress at 7/8 is conclusive.
                if killCurrent >= (killTotal - 1) then
                    qlog(string_format("QUEST COMPLETE (stuck at %d/%d for 15s) — tracker lagging, marking complete",
                        killCurrent, killTotal))
                    LevelState.QuestAccepted     = false
                    LevelState.QuestPhase        = "route"
                    LevelState.LastQuestAt       = 0
                    LevelState.EnemyApproachDone = false
                    LevelState._lastKillLogAt    = nil
                    LevelState._lastKillCount    = nil
                    LevelState._lastKillChangeAt = nil
                    -- CRITICAL: Force re-read level and update route so we pick
                    -- the NEXT quest, not the same one again.
                    local newLevel, _ = readLocalLevel()
                    LevelState.CurrentLevel = newLevel
                    qlog(string_format("Re-reading level after quest complete: Lv %d", newLevel))
                    updateQuestRoute()
                    qlog(string_format("Next route: %s @ %s via %s",
                        tostring(LevelState.RouteEnemy), tostring(LevelState.RouteIsland),
                        tostring(LevelState.RouteQuestGiver)))
                    notifyUser("Quest complete (stuck at " .. killCurrent .. "/" .. killTotal .. ")! Routing to next quest.", 4)
                    return
                end
                -- If we're stuck at a lower count (e.g. 3/8), re-approach the spawn
                qlog(string_format("STUCK at %d/%d for 15s — re-approaching spawn area to find mobs",
                    killCurrent, killTotal))
                LevelState.EnemyApproachDone = false
                LevelState._lastKillChangeAt = os_clock()  -- reset timer
                -- Force a fresh NPC scan with larger range
                local oldRange = Config.Range
                Config.Range = 2000
                scanNPCs()
                Config.Range = oldRange
                qlog(string_format("Fresh scan: %d NPCs in range (expanded to 2000 studs)", #NearbyNPCs))
                return
            end
        end

        -- If we've reached the kill count, the quest is COMPLETE!
        if killTotal > 0 and killCurrent >= killTotal then
            qlog(string_format("Quest COMPLETE! %d/%d kills reached — waiting for tracker to disappear",
                killCurrent, killTotal))
            setStatus(string_format("Quest complete: %d/%d — waiting for turn-in", killCurrent, killTotal))
        end
    end

    -- If quest tracker disappeared, the quest is done — reset and re-route
    if not isQuestTrackerVisible() then
        qlog("Quest tracker disappeared — quest complete, resetting for next route")
        LevelState.QuestAccepted     = false
        LevelState.QuestPhase        = "route"
        LevelState.LastQuestAt       = 0
        LevelState.EnemyApproachDone = false
        LevelState._lastKillLogAt    = nil
        LevelState._lastKillCount    = nil
        LevelState._lastKillChangeAt = nil
        local newLevel, _ = readLocalLevel()
        LevelState.CurrentLevel = newLevel
        qlog(string_format("Re-reading level after tracker disappeared: Lv %d", newLevel))
        updateQuestRoute()
        qlog(string_format("Next route: %s @ %s via %s",
            tostring(LevelState.RouteEnemy), tostring(LevelState.RouteIsland),
            tostring(LevelState.RouteQuestGiver)))
        notifyUser("Quest complete! Routing to next quest.", 4)
        return
    end

    -- If the active quest doesn't match our route, reset so we re-accept
    if not questTrackerMatchesEnemy() then
        qlog("Quest tracker text does NOT match route enemy — re-accepting")
        qlog("  Tracker text: " .. tostring(trackerText):sub(1, 100))
        qlog("  Expected enemy: " .. tostring(LevelState.RouteEnemy))
        LevelState.QuestAccepted     = false
        LevelState.QuestPhase        = "route"
        LevelState.EnemyApproachDone = false
        LevelState._lastKillLogAt    = nil
        LevelState._lastKillCount    = nil
        LevelState._lastKillChangeAt = nil
        return
    end

    -- SMART RESCAN is now handled by the light rescan inside the kill count
    -- block above (scans only Workspace.Enemies[RouteEnemy] for speed).
    -- The fallbacks below still call the full scanNPCs() for periodic refresh
    -- and as a safety net when no matching mob is found.
    local shouldRescan = false

    -- Fallback: rescan every 2 seconds
    if not LevelState._lastFightScanAt or (os_clock() - LevelState._lastFightScanAt) > 2.0 then
        shouldRescan = true
    end

    -- Also rescan if no matching alive mob in the list
    local hasMatchingMob = false
    for _, npc in ipairs(NearbyNPCs) do
        if not isBossEnemy(npc.Name) then
            local matchExact = string_lower(npc.Name) == string_lower(LevelState.RouteEnemy)
            local matchSub = string_find(string_lower(npc.Name), string_lower(LevelState.RouteEnemy), 1, true)
            if (matchExact or matchSub) and type(npc.Health) == "number" and npc.Health > 0 then
                hasMatchingMob = true
                break
            end
        end
    end
    if not hasMatchingMob then
        shouldRescan = true
    end

    if shouldRescan then
        scanNPCs()
        LevelState._lastFightScanAt = os_clock()
    end

    -- Sub-step 3a: Float 30 studs above the enemy spawn position
    if not LevelState.EnemyApproachDone then
        local spawnPos = LevelState.RouteEnemySpawnCFrame
        if not spawnPos then return end

        local approachPos = spawnPos + Vector3_new(0, Config.EnemyApproachHeight, 0)
        if LevelState.TweenSignal and not LevelState.TweenSignal.Completed then
            if LevelState.TweenPurpose == "island" then return end
        end

        if isPlayerNear(approachPos, 8) then
            LevelState.EnemyApproachDone = true
            LevelState.QuestPhase = "fight"
            qlog("Arrived above enemy spawn — scanning for mobs")
            setStatus("Fighting: " .. tostring(LevelState.RouteEnemy))
        else
            local root = getLocalRoot()
            if root then
                qlog("Tweening to enemy spawn (30 studs above)")
                LevelState.TweenSignal  = tweenRootTo(
                    root, approachPos, spawnPos,
                    Config.EnemyApproachSpeed, Config.LevelTweenEasing)
                LevelState.TweenPurpose = "island"
                LevelState.QuestPhase   = "fight"
                setStatus("Approaching enemy spawn area")
            end
            return
        end
    end

    -- Sub-step 3b: Scan workspace.Enemies for our target mob
    -- Skip any NPC that is a boss (safety net — should never happen since
    -- QuestRoutes excludes bosses, but double-check here)
    local target = nil
    local bestDist = nil
    local foundAliveTarget = false
    for _, npc in ipairs(NearbyNPCs) do
        -- Skip bosses entirely
        if not isBossEnemy(npc.Name) then
            if string_lower(npc.Name) == string_lower(LevelState.RouteEnemy)
            or string_find(string_lower(npc.Name), string_lower(LevelState.RouteEnemy), 1, true) then
                if type(npc.Health) == "number" and npc.Health > 0 then
                    foundAliveTarget = true
                    local d = npc.Distance or 0
                    if not target or d < bestDist then
                        target = npc
                        bestDist = d
                    end
                end
            end
        end
    end

    -- ════════════════════════════════════════════════════════════════════
    -- MAKESHIFT QUEST COMPLETION DETECTION:
    -- If the quest tracker text is gone (getQuestTrackerText returns nil)
    -- OR we're at killTotal-1 with no alive target mobs, the quest is done.
    -- ════════════════════════════════════════════════════════════════════

    -- Check 1: Quest tracker text is gone (Matcha returns "failed to fetch text" or empty)
    local currentTrackerText = getQuestTrackerText()
    if not currentTrackerText then
        qlog("QUEST COMPLETE: quest tracker text is gone (nil/empty/failed-to-fetch)")
        LevelState.QuestAccepted     = false
        LevelState.QuestPhase        = "route"
        LevelState.LastQuestAt       = 0
        LevelState.EnemyApproachDone = false
        LevelState._lastKillLogAt    = nil
        LevelState._lastKillCount    = nil
        LevelState._lastKillChangeAt = nil
        local newLevel, _ = readLocalLevel()
        LevelState.CurrentLevel = newLevel
        qlog(string_format("Re-reading level: Lv %d", newLevel))
        updateQuestRoute()
        notifyUser("Quest complete! Routing to next quest.", 4)
        return
    end

    -- Check 2: At killTotal-1 with no alive target mobs → last mob just killed
    if killCurrent and killTotal and killCurrent >= (killTotal - 1) and not foundAliveTarget then
        qlog(string_format("QUEST LIKELY COMPLETE: at %d/%d with no alive %s in range — last mob killed",
            killCurrent, killTotal, tostring(LevelState.RouteEnemy)))
        -- Force a fresh scan to double-check
        scanNPCs()
        local stillNoTarget = true
        for _, npc in ipairs(NearbyNPCs) do
            if not isBossEnemy(npc.Name) then
                if string_find(string_lower(npc.Name), string_lower(LevelState.RouteEnemy), 1, true) then
                    if type(npc.Health) == "number" and npc.Health > 0 then
                        stillNoTarget = false
                        break
                    end
                end
            end
        end
        if stillNoTarget then
            qlog("Confirmed: no alive target mobs — quest is complete!")
            LevelState.QuestAccepted     = false
            LevelState.QuestPhase        = "route"
            LevelState.LastQuestAt       = 0
            LevelState.EnemyApproachDone = false
            LevelState._lastKillLogAt    = nil
            LevelState._lastKillCount    = nil
            LevelState._lastKillChangeAt = nil
            local newLevel, _ = readLocalLevel()
            LevelState.CurrentLevel = newLevel
            qlog(string_format("Re-reading level: Lv %d", newLevel))
            updateQuestRoute()
            notifyUser("Quest complete (last mob killed)! Routing to next quest.", 4)
            return
        end
    end

    if not target then
        LevelState.Target = "none"
        setStatus("Scanning for " .. tostring(LevelState.RouteEnemy) .. "...")
        return
    end

    LevelState.Target = target.Name
    moveToLevelTarget(target)
    setStatus("Killing: " .. tostring(target.Name))
end

-- ============================================================================
-- AUTO-BUSO (Enhancement Haki) BACKGROUND LOOP
-- Automatically enables Buso/Enhancement Haki when fighting mobs.
-- Checks every 2 seconds if Buso is active; if not, presses the key.
-- ============================================================================
task_spawn(function()
    local lastBusoCheck = 0
    while Running do
        if Config.AutoBuso and LevelState.QuestAccepted then
            if os_clock() - lastBusoCheck > 2.0 then
                lastBusoCheck = os_clock()
                -- Check if Buso is already active by looking for the Buso indicator
                -- in the player's character
                local lp = getLocalPlayer()
                local hasBuso = false
                if lp then
                    local char = safeProp(lp, "Character")
                    if char then
                        -- Buso adds a "Buso" attribute or child to the character
                        local buso = safeFind(char, "Buso")
                        if buso then hasBuso = true end
                        -- Also check Humanoid for the Buso state
                        local hum = safeFind(char, "Humanoid")
                        if hum then
                            -- Check for the BusoScreenGui or similar indicator
                            local pg = safeFind(lp, "PlayerGui")
                            if pg then
                                local main = safeFind(pg, "Main")
                                if main then
                                    local busoIndicator = safeFind(main, "Buso")
                                    if busoIndicator and safeProp(busoIndicator, "Visible") ~= false then
                                        hasBuso = true
                                    end
                                end
                            end
                        end
                    end
                end
                if not hasBuso then
                    -- Press J to toggle Buso (default Blox Fruits keybind)
                    if type(keypress) == "function" and type(keyrelease) == "function" then
                        pcall(function() keypress(0x4A) end)  -- J key
                        task_wait(0.05)
                        pcall(function() keyrelease(0x4A) end)
                        qlog("Auto-Buso: pressed J to enable Enhancement Haki")
                    end
                end
            end
        end
        task_wait(1.0)
    end
end)

-- ============================================================================
-- FRUIT ESP BACKGROUND LOOP
-- Scans workspace for Devil Fruit objects and draws simple nametags.
-- 
-- HOW BLOX FRUITS SPAWN FRUITS (verified from multiple sources):
--   1. Primary location: Workspace.DevilFruitSpawner.SpawnedDFS
--      (confirmed by devforum.roblox.com thread)
--   2. Alternate: Workspace.Fruits folder
--      (mentioned by infocheats.net Fruit Sniper docs)
--   3. Fallback: Direct children of Workspace named "Fruit"
--      (mentioned by rscripts.net script snippet)
-- 
-- IDENTIFICATION:
--   - Must be a Model (not Part/Folder/Tool)
--   - Name must be "Fruit" OR contain "Fruit" (case-insensitive)
--   - Must have a child named "Handle" (the visible mesh — universal indicator)
-- 
-- NAME EXTRACTION (via getFruitDisplayName):
--   1. Scan children for any with a known fruit name (Bomb, Spike, Flame, etc.)
--   2. Look for a TextLabel inside Handle showing the fruit name
--   3. Strip "Fruit" from the model name (BombFruit → Bomb)
--   4. Fallback to model name as-is
-- ============================================================================

local fruitLabels = {}
NearbyFruits = {}  -- global so the Debug "Print fruits" button can read it

-- Helper: returns true if `obj` looks like a spawned Devil Fruit.
local function isDevilFruit(obj)
    if not obj then return false end
    local cn = safeProp(obj, "ClassName") or ""
    if cn ~= "Model" then return false end
    local name = safeProp(obj, "Name") or ""
    if #name == 0 then return false end
    local lower = string_lower(name)
    -- Must contain "fruit" in the name (covers "Fruit", "BombFruit", "Dragon Fruit", etc.)
    if not string_find(lower, "fruit", 1, true) then return false end
    -- Must have a Handle child (the universal Devil Fruit indicator in Blox Fruits)
    local handle = safeFind(obj, "Handle")
    if not handle then return false end
    -- Handle should be a BasePart (Part/MeshPart/UnionOperation), not a Folder
    local hcn = safeProp(handle, "ClassName") or ""
    if hcn ~= "Part" and hcn ~= "MeshPart" and hcn ~= "UnionOperation" and hcn ~= "WedgePart" and hcn ~= "TrussPart" then
        return false
    end
    return true
end

-- Helper: scan a folder (or Workspace itself) for Devil Fruits.
-- Appends found fruits to `out` as {Object=, Name=, Key=, Position=, Distance=, Rarity=}.
local function scanFruitsIn(parent, rootPos, out, seen)
    if not parent then return end
    for _, child in ipairs(safeChildren(parent)) do
        local ok, isFruit = pcall(isDevilFruit, child)
        if ok and isFruit then
            local name = safeProp(child, "Name") or "Fruit"
            local key = makeKey(child)
            if not seen[key] then
                local pos = getObjectPosition(child)
                if pos then
                    local dist = rootPos and math_sqrt(distSq(rootPos, pos)) or 0
                    if dist <= Config.FruitRange then
                        local displayName, rarity = getFruitDisplayName(child, name)
                        table.insert(out, {
                            Object   = child,
                            Name     = displayName,
                            RawName  = name,
                            Key      = key,
                            Position = pos,
                            Distance = dist,
                            Rarity   = rarity,
                        })
                        seen[key] = true
                    end
                end
            end
        end
    end
end

-- Public scan function — called by the ESP loop AND by the Debug button.
scanFruits = function()
    local root = getLocalRoot()
    local rootPos = root and safeProp(root, "Position")
    local found = {}
    local seen = {}

    -- Location 1: Workspace.DevilFruitSpawner.SpawnedDFS (primary — devforum-confirmed)
    local spawner = safeFind(Workspace, "DevilFruitSpawner")
    if spawner then
        local spawned = safeFind(spawner, "SpawnedDFS") or safeFind(spawner, "SpawnedDF")
        if spawned then
            scanFruitsIn(spawned, rootPos, found, seen)
        end
    end

    -- Location 2: Workspace.Fruits folder (alternate — infocheats-confirmed)
    local fruitsFolder = safeFind(Workspace, "Fruits")
    if fruitsFolder then
        scanFruitsIn(fruitsFolder, rootPos, found, seen)
    end

    -- Location 3: Direct children of Workspace (fallback — rscripts-confirmed)
    -- Skip the spawner/folders we already scanned above to avoid double work.
    scanFruitsIn(Workspace, rootPos, found, seen)

    NearbyFruits = found
    return found
end

task_spawn(function()
    while Running do
        if Config.FruitNametags then
            pcall(scanFruits)

            local usedFruits = {}
            for _, fruit in ipairs(NearbyFruits) do
                local pos = fruit.Position
                if pos then
                    local ok, screen, onScreen = pcall(WorldToScreen, pos)
                    if ok and screen and onScreen then
                        local key = fruit.Key
                        if not fruitLabels[key] then
                            local t = Drawing.new("Text")
                            t.Visible = false
                            t.Center = true
                            t.Outline = true
                            t.Size = 15
                            t.ZIndex = 85
                            pcall(function() t.Font = Drawing.Fonts.System end)
                            fruitLabels[key] = t
                        end
                        local label = fruitLabels[key]
                        -- Build label text: rarity tag + name + distance
                        local rarityTag = ""
                        if fruit.Rarity == "mythical" then
                            rarityTag = "[MYTH] "
                        elseif fruit.Rarity == "legendary" then
                            rarityTag = "[LEG] "
                        elseif fruit.Rarity == "rare" then
                            rarityTag = "[RARE] "
                        end
                        local fruitText = rarityTag .. fruit.Name .. " Fruit"
                        if Config.FruitShowDistance then
                            fruitText = fruitText .. " [" .. tostring(math_floor(fruit.Distance + 0.5)) .. "]"
                        end
                        label.Text = fruitText
                        label.Color = getFruitColor(fruit.Rarity)
                        label.Position = Vector2_new(math_floor(screen.X + 0.5), math_floor(screen.Y + 0.5))
                        label.Visible = true
                        usedFruits[key] = true
                    end
                end
            end
            -- Hide unused fruit labels
            for key, label in pairs(fruitLabels) do
                if not usedFruits[key] then label.Visible = false end
            end
        else
            for _, label in pairs(fruitLabels) do label.Visible = false end
        end
        task_wait(2.0)
    end
end)

-- ============================================================================
-- SERVER HOP BACKGROUND LOOP
-- When enabled, hops to a new server after all chests are collected
-- or after a configurable delay.
-- ============================================================================
task_spawn(function()
    local lastHop = os_clock()
    while Running do
        if Config.ServerHop then
            -- Only hop if not currently questing and no chests available
            if not Config.LevelProcess and Config.ChestTween then
                local noChests = (#NearbyChests == 0)
                local timeSinceHop = os_clock() - lastHop
                if noChests and timeSinceHop > Config.ServerHopDelay then
                    qlog("Server hop: no chests available, hopping to new server")
                    notifyUser("Server hopping...", 3)
                    -- Use TeleportService to hop to a different server
                    pcall(function()
                        local TS = game:GetService("TeleportService")
                        local placeId = game.PlaceId
                        TS:Teleport(placeId, getLocalPlayer())
                    end)
                    lastHop = os_clock()
                    task_wait(10)  -- wait for teleport
                end
            end
        end
        task_wait(5.0)
    end
end)

-- ============================================================================
-- AUTOCLICKER BACKGROUND LOOP
-- Runs as a separate task_spawn — clicks continuously while:
--   1. Config.LevelAutoClick is ON
--   2. LevelState.QuestAccepted is true (quest is active)
--   3. Not currently in a long-distance tween
-- This is SEPARATE from fightStep so it runs every frame regardless of
-- what fightStep is doing (scanning, tweening, etc).
-- ============================================================================
task_spawn(function()
    while Running do
        if Config.LevelAutoClick and LevelState.QuestAccepted then
            if not LevelState.TweenSignal or LevelState.TweenSignal.Completed then
                -- Just burst-click at current mouse position (don't move mouse)
                for i = 1, 5 do
                    if type(mouse1press) == "function" and type(mouse1release) == "function" then
                        pcall(function() mouse1press() end)
                        task_wait(0.04)
                        pcall(function() mouse1release() end)
                        task_wait(0.02)
                    elseif type(mouse1click) == "function" then
                        pcall(function() mouse1click() end)
                        task_wait(0.03)
                    end
                end

                -- Wait for the click interval before next burst
                task_wait(Config.LevelClickInterval)
            else
                task_wait(0.1)  -- tween in progress, wait
            end
        else
            task_wait(0.5)  -- autoclick off or quest not active, slow poll
        end
    end
end)

levelProcessStep = function()
    if not Config.LevelProcess then return end

    local now = os_clock()
    if now - LevelState.LastStepAt < Config.LevelDelay then return end
    LevelState.LastStepAt = now
    LevelState.StepCount  = LevelState.StepCount + 1

    -- Always re-read level + update route
    local currentLevel, levelSource = readLocalLevel()
    LevelState.CurrentLevel = currentLevel
    LevelState.LevelSource  = levelSource
    updateQuestRoute()

    -- Run quest giver step (handles Step 1 + Step 2 of the flow)
    if questGiverStep(now) then
        releaseClickIfNeeded()
        return
    end

    -- If we're in Quest mode and the quest isn't accepted yet, don't fight
    if Config.LevelMode == "Quest" and not LevelState.QuestAccepted then
        return
    end

    -- Make sure NearbyNPCs is fresh
    if #NearbyNPCs < 1 then scanNPCs() end

    -- Run fight step (Step 3 of the flow)
    if Config.LevelMode == "Quest" then
        fightStep()
        return
    end

    -- Manual mode: just pick the closest NPC matching LevelTarget and kill it
    local best = nil
    local bestDistance = nil
    for _, npc in ipairs(NearbyNPCs) do
        local matches = true
        if type(Config.LevelTarget) == "string" and #Config.LevelTarget > 0 then
            matches = string_find(string_lower(npc.Name), string_lower(Config.LevelTarget), 1, true) ~= nil
        end
        if matches and type(npc.Health) == "number" and npc.Health > 0 then
            local d = npc.Distance or 0
            if not best or d < bestDistance then
                best = npc
                bestDistance = d
            end
        end
    end

    if not best then
        LevelState.Target = "none"
        setStatus("Level: no target")
        return
    end

    LevelState.Target = best.Name
    moveToLevelTarget(best)
    if not LevelState.TweenSignal or LevelState.TweenSignal.Completed then
        clickLevelTarget()
    end
    releaseClickIfNeeded()
    setStatus("Level: " .. tostring(best.Name))
end

-- ============================================================================
-- 19. CHEST TWEEN (unchanged from original, kept for completeness)
-- ============================================================================

nameMatchesChest = function(name)
    if type(Config.ChestFilter) ~= "string" or #Config.ChestFilter < 1 then return true end
    local lowerName = string_lower(tostring(name or ""))
    local lowerFilter = string_lower(Config.ChestFilter)
    return string_find(lowerName, lowerFilter, 1, true) ~= nil
end

scanChests = function()
    local root = getLocalRoot()
    local rootPos = root and safeProp(root, "Position")
    local found = {}
    local seen = {}

    -- In Blox Fruits, chests are Part instances directly in Workspace.
    -- When collected, they are DESTROYED (Parent becomes nil).
    -- To detect available chests: just check if they exist in Workspace.
    -- No transparency or property checks needed.

    -- Chest types:
    --   Chest1 = Silver/Brown chest
    --   Chest2 = Gold chest
    --   Chest3 = Blue/Diamond chest

    local function tryAddChest(obj, name)
        -- Check if the chest still exists (not destroyed/collected)
        local parent = safeProp(obj, "Parent")
        if not parent then return end

        local pos = getObjectPosition(obj)
        local key = makeKey(obj)
        if pos and not seen[key] then
            local distance = rootPos and math_sqrt(distSq(rootPos, pos)) or 0
            if distance <= Config.ChestRange then
                -- Display name based on actual chest name
                local displayName = name
                local lowerName = string_lower(name)
                if string_find(lowerName, "silver", 1, true) or name == "Chest1" then
                    displayName = "Silver Chest"
                elseif string_find(lowerName, "gold", 1, true) or name == "Chest2" then
                    displayName = "Gold Chest"
                elseif string_find(lowerName, "diamond", 1, true) or name == "Chest3" then
                    displayName = "Diamond Chest"
                end
                table.insert(found, {
                    Object   = obj,
                    Name     = displayName,
                    Key      = key,
                    Position = pos,
                    Distance = distance,
                })
                seen[key] = true
            end
        end
    end

    -- Chests are inside Workspace.ChestModels folder, named:
    --   SilverChest (was Chest1)
    --   GoldChest (was Chest2)
    --   DiamondChest (was Chest3)
    -- Also check direct workspace children as fallback.
    local chestFolder = safeFind(Workspace, "ChestModels")
    if chestFolder then
        for _, child in ipairs(safeChildren(chestFolder)) do
            local name = safeProp(child, "Name") or ""
            local lowerName = string_lower(name)
            if string_find(lowerName, "chest", 1, true) then
                tryAddChest(child, name)
            end
        end
    end

    -- Fallback: also check direct workspace children (for Chest1/2/3 naming)
    for _, child in ipairs(safeChildren(Workspace)) do
        local name = safeProp(child, "Name") or ""
        if name == "Chest1" or name == "Chest2" or name == "Chest3" then
            tryAddChest(child, name)
        end
    end

    NearbyChests = found
    ChestState.Count = #NearbyChests
    ChestState.LastScanAt = os_clock()
end

selectChestTarget = function()
    local now = os_clock()
    local best = nil
    local bestDistance = nil
    for _, chest in ipairs(NearbyChests) do
        local lastVisit = ChestVisited[chest.Key] or 0
        if now - lastVisit >= Config.ChestCooldown then
            local distance = chest.Distance or 0
            if not best or distance < bestDistance then
                best = chest
                bestDistance = distance
            end
        end
    end
    return best
end

chestProcessStep = function()
    -- Only scan/tween chests if chest features are enabled
    if not Config.ChestTween and not Config.ChestESP then return end

    -- Scan every 1 second (cheap — just 3 FindFirstChild calls)
    if os_clock() - ChestState.LastScanAt > 1.0 then
        scanChests()
    end

    -- Render chest ESP (like NPC ESP — simple labels with name + distance)
    if Config.ChestESP then
        local usedChests = {}
        for _, chest in ipairs(NearbyChests) do
            local pos = chest.Position
            if pos then
                local ok, screen, onScreen = pcall(WorldToScreen, pos)
                if ok and screen and onScreen then
                    local cx = math_floor(screen.X + 0.5)
                    local cy = math_floor(screen.Y + 0.5)
                    if not chestLabels[chest.Key] then
                        local t = Drawing.new("Text")
                        t.Visible = false
                        t.Center = true
                        t.Outline = true
                        t.Size = 13
                        t.ZIndex = 82
                        pcall(function() t.Font = Drawing.Fonts.System end)
                        chestLabels[chest.Key] = t
                    end
                    local label = chestLabels[chest.Key]
                    label.Text = chest.Name .. " [" .. tostring(math_floor(chest.Distance + 0.5)) .. "]"
                    -- Color by chest type
                    if chest.Name == "Gold Chest" then
                        label.Color = Color3.fromRGB(255, 215, 0)
                    elseif chest.Name == "Diamond Chest" then
                        label.Color = Color3.fromRGB(85, 255, 255)
                    elseif chest.Name == "Silver Chest" then
                        label.Color = Color3.fromRGB(180, 180, 180)
                    else
                        label.Color = Color3.fromRGB(255, 200, 50)
                    end
                    label.Position = Vector2_new(cx, cy)
                    label.Visible = true
                    usedChests[chest.Key] = true
                end
            end
        end
        -- Hide unused chest labels
        for key, label in pairs(chestLabels) do
            if not usedChests[key] then label.Visible = false end
        end
    else
        for _, label in pairs(chestLabels) do label.Visible = false end
    end

    -- Only do chest TWEENING if enabled AND not questing
    if not Config.ChestTween then return end
    if Config.LevelProcess then return end  -- pause while questing

    if ChestState.TweenSignal and not ChestState.TweenSignal.Completed then return end
    if ChestState.TweenSignal and ChestState.TweenSignal.Completed then
        ChestVisited[ChestState.TargetKey] = os_clock()
        ChestState.LastVisitAt = os_clock()
        ChestState.TweenSignal = nil
        setStatus("Chest reached: " .. tostring(ChestState.Target))
        return
    end

    local chest = selectChestTarget()
    if not chest then
        ChestState.Target = "none"
        setStatus("Chest: no target")
        return
    end

    local root = getLocalRoot()
    local targetPos = chest.Position + Vector3_new(0, Config.ChestHeightOffset, 0)
    if not root or not targetPos then return end

    ChestState.Target    = chest.Name
    ChestState.TargetKey = chest.Key
    ChestState.TweenSignal = tweenRootTo(root, targetPos, chest.Position, Config.ChestTweenSpeed, Config.LevelTweenEasing)
    setStatus("Chest: " .. tostring(chest.Name))
end

-- ============================================================================
-- 20. CLEANUP
-- ============================================================================

function cleanup()
    if Cleaned then return end
    Cleaned = true
    Running = false
    forceReleaseClick()
    restoreAllHeads()
    removeAllDrawings()
    pcall(function() QuestHUD:Remove() end)
    pcall(function() QuestHUD2:Remove() end)
    pcall(function() QuestHUD3:Remove() end)
    for _, l in ipairs(rangeCircleLines) do pcall(function() l:Remove() end) end
    for _, label in pairs(chestLabels) do pcall(function() label:Remove() end) end
    pcall(function() UILib:Unload() end)
    _G.__BloxFruitsCompanion_Cleanup = nil
end

_G.__BloxFruitsCompanion_Cleanup = cleanup

-- ============================================================================
-- 21. UI BUILD
-- ============================================================================

pcall(function()
    UILib:SetMenuTitle(Config.Title)
    UILib:SetWatermarkEnabled(Config.Watermark)
    UILib:SetMenuSize(Vector2_new(720, 560))
    UILib:CenterMenu()
    UILib._theming.accent = Config.Accent
    UILib._theming.text   = Config.TextColor
end)

-- Activity string in watermark
pcall(function()
    UILib:RegisterActivity(function()
        local uptime = math_floor(os_clock() - StartedAt)

        -- Read player health
        local hpText = ""
        local lp = getLocalPlayer()
        if lp then
            local char = safeProp(lp, "Character")
            if char then
                local hum = safeFind(char, "Humanoid")
                if hum then
                    local hp = safeProp(hum, "Health")
                    local mhp = safeProp(hum, "MaxHealth")
                    if type(hp) == "number" and type(mhp) == "number" then
                        hpText = "HP " .. math_floor(hp) .. "/" .. math_floor(mhp) .. " | "
                    end
                end
            end
        end

        -- Read money (Beli)
        local moneyText = ""
        if lp then
            local data = safeFind(lp, "Data")
            if data then
                local money = safeFind(data, "Beli")
                if money then
                    local val = readValueObjectNumber(money)
                    if val then
                        -- Format with commas: 1234567 -> 1,234,567
                        local moneyStr = tostring(math_floor(val))
                        local formatted = ""
                        while #moneyStr > 3 do
                            formatted = "," .. moneyStr:sub(-3) .. formatted
                            moneyStr = moneyStr:sub(1, -4)
                        end
                        formatted = moneyStr .. formatted
                        moneyText = "$" .. formatted .. " | "
                    end
                end
            end
        end

        -- Format uptime as H:MM:SS or M:SS
        local function formatTime(seconds)
            local h = math_floor(seconds / 3600)
            local m = math_floor((seconds % 3600) / 60)
            local s = seconds % 60
            if h > 0 then
                return string_format("%d:%02d:%02d", h, m, s)
            else
                return string_format("%d:%02d", m, s)
            end
        end
        local timeText = formatTime(uptime)

        if Config.ChestTween then
            return hpText .. moneyText ..
                   "Chests " .. tostring(ChestState.Count) ..
                   " | " .. timeText
        end

        -- Quest progress
        local questText = ""
        local trackerText = getQuestTrackerText()
        if trackerText and #trackerText > 5 then
            questText = " | " .. trackerText:sub(1, 40)
        end

        return hpText .. moneyText ..
               "Lv " .. tostring(LevelState.CurrentLevel) ..
               " | " .. tostring(LevelState.RouteEnemy) ..
               " @ " .. tostring(LevelState.RouteIsland) ..
               " | " .. tostring(LevelState.QuestPhase) ..
               questText ..
               " | " .. timeText
    end)
end)

-- Main tab
local mainTab = UILib:Tab("Main")

local levelSection = mainTab:Section("Level Farm")
levelSection:Toggle("Level process", Config.LevelProcess, function(v)
    Config.LevelProcess = v == true
    if Config.LevelProcess then
        notifyUser("Level farm started", 4)
    else
        notifyUser("Level farm paused", 4)
        forceReleaseClick()
    end
end)
levelSection:Toggle("Auto click (attack)", Config.LevelAutoClick, function(v)
    Config.LevelAutoClick = v == true
    if not Config.LevelAutoClick then forceReleaseClick() end
    if Config.LevelAutoClick then
        notifyUser("Autoclicker ON — will burst-click enemies every 0.3s", 4)
    else
        notifyUser("Autoclicker OFF", 3)
    end
end)
levelSection:Toggle("Smart rescan", Config.SmartRescan, function(v)
    Config.SmartRescan = v == true
    if Config.SmartRescan then
        notifyUser("Smart rescan ON — light rescan of target enemy folder on each kill (+1)", 3)
    else
        notifyUser("Smart rescan OFF — uses timed scans only", 3)
    end
end)
levelSection:Toggle("Auto quest route", Config.LevelAutoRoute, function(v)
    Config.LevelAutoRoute = v == true
    updateQuestRoute()
end)
levelSection:Dropdown("Level mode", {Config.LevelMode}, {"Quest", "Manual"}, false, function(v)
    if type(v) == "table" and v[1] then
        Config.LevelMode = v[1]
        setStatus("Level mode: " .. Config.LevelMode)
    end
end)
levelSection:Textbox("Manual target", Config.LevelTarget, function(v)
    Config.LevelTarget = v or ""
end)
levelSection:Slider("Step delay", Config.LevelDelay, 0.01, 0.01, 1.0, " sec", function(v)
    Config.LevelDelay = v
end)
levelSection:Toggle("Move to target", Config.LevelMoveToTarget, function(v)
    Config.LevelMoveToTarget = v == true
end)
levelSection:Slider("Kill hover height", Config.LevelHeightOffset, 1, -20, 40, " studs", function(v)
    Config.LevelHeightOffset = v
end)
levelSection:Slider("Kill hover dist", Config.LevelDistance, 1, 0, 30, " studs", function(v)
    Config.LevelDistance = v
end)
levelSection:Slider("Tween speed", Config.LevelTweenSpeed, 10, 25, 5000, " studs/s", function(v)
    Config.LevelTweenSpeed = v
end)
levelSection:Dropdown("Tween easing", {Config.LevelTweenEasing},
    {"linear", "smoothstep", "ease_in_quad", "ease_out_quad", "ease_in_out_quad"},
    false, function(v)
        if type(v) == "table" and v[1] then
            Config.LevelTweenEasing = v[1]
        end
    end)
levelSection:Slider("Max tween speed", Config._maxTweenSpeed or 500, 10, 100, 5000, " studs/s", function(v)
    Config._maxTweenSpeed = v
    if Config.LevelTweenSpeed > v then Config.LevelTweenSpeed = v end
    if Config.TeleportTweenSpeed > v then Config.TeleportTweenSpeed = v end
    if Config.ChestTweenSpeed > v then Config.ChestTweenSpeed = v end
    if Config.EnemyApproachSpeed > v then Config.EnemyApproachSpeed = v end
end)

local clickSection = mainTab:Section("Autoclicker")
clickSection:Slider("Click hold", Config.LevelClickHold, 0.01, 0.01, 0.35, " sec", function(v)
    Config.LevelClickHold = v
end)
clickSection:Slider("Click interval", Config.LevelClickInterval, 0.01, 0.01, 1.0, " sec", function(v)
    Config.LevelClickInterval = v
end)
clickSection:Button("Test click", function()
    local old = Config.LevelAutoClick
    Config.LevelAutoClick = true
    LevelState.LastClickAt = 0
    if clickLevelTarget() then
        notifyUser("Test click sent", 3)
    else
        notifyUser("Test click skipped", 3)
    end
    Config.LevelAutoClick = old
end)

-- Quest giver tab
-- NPCs tab created BEFORE Quest Giver so it appears second in the UI
local npcTab = UILib:Tab("NPCs")
local tagSection = npcTab:Section("NPC Nametags")
local npcTagToggle = tagSection:Toggle("NPC nametags", Config.NpcNametags, function(v)
    Config.NpcNametags = v == true
    if not Config.NpcNametags then hideAllDrawings() end
end)
npcTagToggle:AddColorpicker("Name color", Config.Accent, true, function(c)
    if c then Config.Accent = c end
end)
tagSection:Toggle("Box ESP", Config.NpcBoxESP, function(v)
    Config.NpcBoxESP = v == true
    if not Config.NpcBoxESP then hideAllBoxes() end
end)
tagSection:Toggle("Healthbars", Config.NpcHealthbars, function(v)
    Config.NpcHealthbars = v == true
    if not Config.NpcHealthbars then hideAllDrawings() end
end)
tagSection:Toggle("Dynamic health color", Config.DynamicHealthColor, function(v)
    Config.DynamicHealthColor = v == true
end)
tagSection:Toggle("Static health color", not Config.DynamicHealthColor, function(v)
    Config.DynamicHealthColor = not (v == true)
end)
local staticHealthColorToggle = tagSection:Toggle("Static bar color picker", false, function(v)
    -- placeholder — colorpicker added below
end)
staticHealthColorToggle:AddColorpicker("Bar color", Config.StaticHealthColor or Color3.fromRGB(80, 255, 120), true, function(c)
    Config.StaticHealthColor = c
end)
tagSection:Toggle("Show distance", Config.ShowDistance, function(v)
    Config.ShowDistance = v == true
end)
tagSection:Toggle("Health text", Config.ShowHealthText, function(v)
    Config.ShowHealthText = v == true
end)
tagSection:Slider("NPC range", Config.Range, 50, 50, 2000, " studs", function(v)
    Config.Range = v
end)
tagSection:Textbox("NPC filter", Config.NpcFilter, function(v)
    Config.NpcFilter = v or ""
end)

local hitboxSection = npcTab:Section("Head Hitbox")
hitboxSection:Toggle("Head hitbox", Config.HeadHitbox, function(v)
    Config.HeadHitbox = v == true
    if Config.HeadHitbox then applyHitboxesToNearby() else restoreAllHeads() end
end)
hitboxSection:Slider("Head size", Config.HeadSize, 2, 2, 75, " studs", function(v)
    Config.HeadSize = v
    if Config.HeadHitbox then applyHitboxesToNearby() end
end)
hitboxSection:Button("Restore heads", function()
    restoreAllHeads()
    Config.HeadHitbox = false
    notifyUser("Heads restored", 3)
end)

local questTab = UILib:Tab("Quest Giver")
local questSection = questTab:Section("Quest Settings")
questSection:Toggle("Use quest giver", Config.QuestGiverEnabled, function(v)
    Config.QuestGiverEnabled = v == true
    if not Config.QuestGiverEnabled then
        LevelState.QuestPhase = "disabled"
    end
end)
questSection:Toggle("Detect GUI buttons", Config.QuestGuiDetect, function(v)
    Config.QuestGuiDetect = v == true
end)
questSection:Textbox("Giver override", Config.QuestGiverOverride, function(v)
    Config.QuestGiverOverride = v or ""
    LevelState.QuestAccepted = false
    LevelState.QuestPhase = "route"
end)
questSection:Slider("Giver distance", Config.QuestGiverDistance, 1, 2, 30, " studs", function(v)
    Config.QuestGiverDistance = v
end)
questSection:Slider("Quest refresh", Config.QuestRefreshTime, 5, 5, 300, " sec", function(v)
    Config.QuestRefreshTime = v
end)
questSection:Slider("Option button scan", Config.QuestOptionClickTime, 1, 1, 15, " sec", function(v)
    Config.QuestOptionClickTime = v
end)
questSection:Slider("Confirm delay", Config.QuestConfirmDelay, 0.1, 0.1, 6.0, " sec", function(v)
    Config.QuestConfirmDelay = v
end)
questSection:Slider("Confirm scan time", Config.QuestConfirmScanTime, 1, 1, 15, " sec", function(v)
    Config.QuestConfirmScanTime = v
end)

local questDiagSection = questTab:Section("Diagnostics")
questDiagSection:Button("Find quest giver", function()
    updateQuestRoute()
    local giver, _ = findQuestGiver()
    if giver then
        LevelState.QuestGiverObject = safeProp(giver, "Name") or getWantedQuestGiver()
        notifyUser("Found: " .. tostring(LevelState.QuestGiverObject), 4)
    else
        notifyUser("Missing: " .. tostring(getWantedQuestGiver()), 4)
    end
end)
questDiagSection:Button("Show quest route", function()
    updateQuestRoute()
    notifyUser("Route: Lv " .. tostring(LevelState.CurrentLevel) ..
               " -> " .. tostring(LevelState.RouteEnemy) ..
               " @ " .. tostring(LevelState.RouteIsland) ..
               " via " .. tostring(LevelState.RouteQuestGiver), 6)
end)
questDiagSection:Button("Check dialog open", function()
    if isQuestDialogOpen() then
        notifyUser("Dialog IS open", 3)
    else
        notifyUser("Dialog is NOT open", 3)
    end
end)
questDiagSection:Button("Dump GUI text", function()
    local count = dumpAllGuiText()
    notifyUser("Dumped " .. tostring(count) .. " text objects to F9 console", 5)
    if type(setclipboard) == "function" then
        local log = table.concat(LevelState.QuestLog, "\n")
        pcall(setclipboard, log)
        notifyUser("Quest log also copied to clipboard", 4)
    end
end)
questDiagSection:Button("Find Gorilla button", function()
    updateQuestRoute()
    local x, y, obj = findQuestGuiButton(LevelState.RouteEnemy)
    if x and y then
        notifyUser("Found button for " .. tostring(LevelState.RouteEnemy) ..
                   " at (" .. tostring(x) .. ", " .. tostring(y) .. ")", 6)
        qlog(string_format("MANUAL FIND: button for %s at (%d, %d) obj=%s",
            tostring(LevelState.RouteEnemy), x, y, safeFullName(obj):sub(1, 120)))
    else
        notifyUser("NO button found for " .. tostring(LevelState.RouteEnemy) ..
                   " — dumped GUI to console", 6)
        dumpAllGuiText()
    end
end)
questDiagSection:Button("List option buttons (v2)", function()
    local dialog = findDialogContainer()
    if not dialog then
        notifyUser("No dialog container found — is the dialog open?", 5)
        qlog("MANUAL LIST: no dialog container found")
        return
    end
    qlog("MANUAL LIST: dialog container = " .. safeFullName(dialog):sub(1, 120))
    local buttons = getOptionButtonsInDialog(dialog)
    qlog(string_format("MANUAL LIST: %d option buttons in dialog:", #buttons))
    for i, b in ipairs(buttons) do
        qlog(string_format("  [%d] text=%q pos=(%d,%d) path=%s",
            i, b.text:sub(1, 40), b.x, b.y, b.path))
    end
    notifyUser("Found " .. tostring(#buttons) .. " option buttons (see F9 console)", 5)
end)
questDiagSection:Button("Click option 2 (v2 test)", function()
    updateQuestRoute()
    local ok = clickOptionButtonByIndex(LevelState.RouteOption)
    if ok then
        notifyUser("Clicked option " .. tostring(LevelState.RouteOption) .. " via v2 scanner", 4)
    else
        notifyUser("v2 click FAILED — see F9 console", 5)
    end
end)
questDiagSection:Button("Find Confirm button", function()
    local x, y, obj = findQuestConfirmButtonStrict()
    if x and y then
        notifyUser("Found Confirm at (" .. tostring(x) .. ", " .. tostring(y) .. ")", 6)
        qlog(string_format("MANUAL FIND CONFIRM: at (%d, %d) obj=%s",
            x, y, safeFullName(obj):sub(1, 120)))
    else
        notifyUser("NO Confirm button found — dumped GUI to console", 6)
        dumpAllGuiText()
    end
end)
questDiagSection:Button("Test click at screen center", function()
    local vpX, vpY = getViewportSize()
    local x, y = math_floor(vpX / 2), math_floor(vpY / 2)
    qlog(string_format("TEST CLICK: moving mouse to (%d, %d) and clicking", x, y))
    guiClickAt(x, y, { holdTime = 0.08, settleTime = 0.15 })
    notifyUser("Test-clicked at (" .. tostring(x) .. ", " .. tostring(y) .. ")", 4)
end)
questDiagSection:Button("Test click at Gorillas pos", function()
    -- Hardcoded test: click at (1483, 587) which is where Option2 "Gorillas"
    -- was in the user's log. User can verify if the click actually lands on
    -- the Gorillas button when the dialog is open.
    local x, y = 1483, 587
    local vpX, vpY = getViewportSize()
    qlog(string_format("TEST GORILLA CLICK: viewport=%dx%d, clicking at (%d, %d)", vpX, vpY, x, y))
    if x > vpX or y > vpY then
        notifyUser(string.format("WARNING: (%d,%d) is outside viewport %dx%d — won't work", x, y, vpX, vpY), 6)
    end
    guiClickAt(x, y, { holdTime = 0.10, settleTime = 0.20 })
    notifyUser("Clicked at Gorillas position (1483, 587) — did quest accept?", 5)
end)
questDiagSection:Button("Force re-find dialog", function()
    invalidateDialogCache()
    local d = findDialogContainer({ forceRefresh = true })
    if d then
        notifyUser("Dialog container: " .. safeFullName(d):sub(1, 60), 5)
    else
        notifyUser("No dialog container found — is dialog open?", 5)
    end
end)
questDiagSection:Button("Test mouse variant 1 (x,y)", function()
    local vpX, vpY = getViewportSize()
    local x, y = math_floor(vpX * 0.75), math_floor(vpY * 0.30)
    qlog(string_format("TEST MOUSE v1 (noextra): mousemoveabs(%d, %d)", x, y))
    local ok = pcall(function() mousemoveabs(x, y) end)
    qlog(string_format("TEST MOUSE v1 result: %s", tostring(ok)))
    notifyUser("v1 (x,y) tried — did mouse move to top-right?", 5)
end)
questDiagSection:Button("Test mouse variant 2 (0,x,y)", function()
    local vpX, vpY = getViewportSize()
    local x, y = math_floor(vpX * 0.75), math_floor(vpY * 0.30)
    qlog(string_format("TEST MOUSE v2 (leading0): mousemoveabs(0, %d, %d)", x, y))
    local ok = pcall(function() mousemoveabs(0, x, y) end)
    qlog(string_format("TEST MOUSE v2 result: %s", tostring(ok)))
    notifyUser("v2 (0,x,y) tried — did mouse move to top-right?", 5)
end)
questDiagSection:Button("Test mouse variant 3 (x,y,0)", function()
    local vpX, vpY = getViewportSize()
    local x, y = math_floor(vpX * 0.75), math_floor(vpY * 0.30)
    qlog(string_format("TEST MOUSE v3 (trailing0): mousemoveabs(%d, %d, 0)", x, y))
    local ok = pcall(function() mousemoveabs(x, y, 0) end)
    qlog(string_format("TEST MOUSE v3 result: %s", tostring(ok)))
    notifyUser("v3 (x,y,0) tried — did mouse move to top-right?", 5)
end)
questDiagSection:Button("Force mouse variant (cycle)", function()
    -- Force the cached variant to next one, so user can manually try each
    if _mousemoveabs_variant == "noextra" then
        _mousemoveabs_variant = "leading0"
        notifyUser("Forced variant: leading0 (0,x,y)", 4)
    elseif _mousemoveabs_variant == "leading0" then
        _mousemoveabs_variant = "trailing0"
        notifyUser("Forced variant: trailing0 (x,y,0)", 4)
    else
        _mousemoveabs_variant = "noextra"
        notifyUser("Forced variant: noextra (x,y)", 4)
    end
    qlog("Forced _mousemoveabs_variant = " .. tostring(_mousemoveabs_variant))
end)
questDiagSection:Button("Check quest active", function()
    if isQuestTrackerVisible() then
        local text = getQuestTrackerText() or "(no text)"
        notifyUser("Quest active: " .. text:sub(1, 60), 6)
    else
        notifyUser("No active quest", 3)
    end
end)
questDiagSection:Button("Reset quest state", function()
    LevelState.QuestAccepted     = false
    LevelState.QuestPhase        = "route"
    LevelState.LastQuestAt       = 0
    LevelState.EnemyApproachDone = false
    LevelState.LastAcceptFailAt  = 0
    LevelState.AcceptFailCount   = 0
    LevelState.AutoPaused        = false
    LevelState.TweenPurpose      = "none"
    notifyUser("Quest state reset (fail counter cleared)", 3)
end)
questDiagSection:Button("Dump quest log", function()
    local n = #LevelState.QuestLog
    if n < 1 then
        notifyUser("Quest log is empty", 3)
        return
    end
    local combined = table.concat(LevelState.QuestLog, "\n")
    if type(setclipboard) == "function" then
        pcall(setclipboard, combined)
        notifyUser("Quest log (" .. tostring(n) .. " entries) copied to clipboard", 5)
    else
        notifyUser("Quest log has " .. tostring(n) .. " entries — see console", 5)
    end
    print("=== QUEST LOG DUMP ===")
    print(combined)
    print("=== END DUMP ===")
end)

-- NPCs tab
-- Teleports tab
local teleportTab = UILib:Tab("Teleports")
local tpSettings = teleportTab:Section("Tween Settings")
tpSettings:Slider("Teleport speed", Config.TeleportTweenSpeed, 10, 50, 5000, " studs/s", function(v)
    Config.TeleportTweenSpeed = v
end)
tpSettings:Slider("Height offset", Config.TeleportHeightOffset, 1, 0, 80, " studs", function(v)
    Config.TeleportHeightOffset = v
end)
tpSettings:Dropdown("Teleport easing", {Config.LevelTweenEasing},
    {"linear", "smoothstep", "ease_in_quad", "ease_out_quad", "ease_in_out_quad"},
    false, function(v)
        if type(v) == "table" and v[1] then
            Config.LevelTweenEasing = v[1]
        end
    end)
tpSettings:Slider("Max tween speed", Config._maxTweenSpeed or 500, 10, 100, 5000, " studs/s", function(v)
    Config._maxTweenSpeed = v
    if Config.LevelTweenSpeed > v then Config.LevelTweenSpeed = v end
    if Config.TeleportTweenSpeed > v then Config.TeleportTweenSpeed = v end
    if Config.ChestTweenSpeed > v then Config.ChestTweenSpeed = v end
    if Config.EnemyApproachSpeed > v then Config.EnemyApproachSpeed = v end
end)
tpSettings:Button("Stop tween", function()
    if LevelState.TweenSignal then
        LevelState.TweenSignal.Completed = true
    end
    LevelState.TweenPurpose = "none"
    notifyUser("Tween stopped", 3)
end)

-- Build a button per island waypoint (sorted)
function addIslandButtons(section, islands)
    for _, name in ipairs(islands) do
        local wp = IslandWaypoints[name]
        if wp then
            section:Button(name, function()
                tweenToWaypoint(name, wp)
            end)
        end
    end
end

local sea1Section = teleportTab:Section("Sea 1")
addIslandButtons(sea1Section, {
    "Starter Island", "Jungle", "Pirate Village", "Desert",
    "Frozen Village", "Marine Fortress", "Skylands", "Prison",
    "Colosseum", "Magma Village", "Underwater City", "Upper Skylands",
    "Fountain City",
})

local sea2Section = teleportTab:Section("Sea 2")
addIslandButtons(sea2Section, {
    "Kingdom of Rose", "Green Zone", "Graveyard Island", "Snow Mountain",
    "Hot and Cold", "Cursed Ship", "Ice Castle", "Forgotten Island",
})

local sea3Section = teleportTab:Section("Sea 3")
addIslandButtons(sea3Section, {
    "Port Town", "Hydra Island", "Great Tree", "Floating Turtle",
    "Haunted Castle", "Sea of Treats", "Tiki Outpost",
})

-- Chests tab
local chestTab = UILib:Tab("Chests")
local chestSection = chestTab:Section("Chest Tween")
chestSection:Toggle("Tween to chests", Config.ChestTween, function(v)
    Config.ChestTween = v == true
    if Config.ChestTween then
        scanChests()
        notifyUser("Chest tween enabled", 4)
    else
        ChestState.TweenSignal = nil
        notifyUser("Chest tween paused", 4)
    end
end)
chestSection:Toggle("Chest ESP", Config.ChestESP, function(v)
    Config.ChestESP = v == true
end)
chestSection:Toggle("Visualize range", Config.ChestVisualRange, function(v)
    Config.ChestVisualRange = v == true
    if not Config.ChestVisualRange then
        -- Hide range circle
        for _, l in ipairs(rangeCircleLines) do pcall(function() l.Visible = false end) end
    end
end)
chestSection:Textbox("Chest filter", Config.ChestFilter, function(v)
    Config.ChestFilter = v or ""
end)
chestSection:Slider("Chest range", Config.ChestRange, 100, 100, 15000, " studs", function(v)
    Config.ChestRange = v
end)
chestSection:Slider("Chest speed", Config.ChestTweenSpeed, 5, 25, 5000, " studs/s", function(v)
    Config.ChestTweenSpeed = v
end)
chestSection:Slider("Chest height", Config.ChestHeightOffset, 1, -10, 30, " studs", function(v)
    Config.ChestHeightOffset = v
end)
chestSection:Slider("Visit cooldown", Config.ChestCooldown, 1, 1, 120, " sec", function(v)
    Config.ChestCooldown = v
end)
chestSection:Button("Clear visited", function()
    ChestVisited = {}
    notifyUser("Chest visit cache cleared", 3)
end)
chestSection:Button("Debug chests", function()
    scanChests()
    qlog("=== CHEST DEBUG ===")
    qlog("ChestESP: " .. tostring(Config.ChestESP))
    qlog("ChestTween: " .. tostring(Config.ChestTween))
    qlog("ChestRange: " .. tostring(Config.ChestRange))
    qlog("ChestFilter: " .. tostring(Config.ChestFilter))
    qlog("NearbyChests count: " .. tostring(#NearbyChests))
    qlog("ChestState.Count: " .. tostring(ChestState.Count))
    qlog("ChestState.Target: " .. tostring(ChestState.Target))
    qlog("ChestState.LastScanAt: " .. tostring(ChestState.LastScanAt))
    qlog("Workspace children scan:")
    local wsCount = 0
    local chestCount = 0
    for _, child in ipairs(safeChildren(Workspace)) do
        wsCount = wsCount + 1
        local name = safeProp(child, "Name") or ""
        local cn = safeProp(child, "ClassName") or "?"
        -- Log EVERY workspace child so we can see what's there
        qlog(string_format("  [%d] %s [%s]", wsCount, name, cn))
        if name == "Chest1" or name == "Chest2" or name == "Chest3" then
            chestCount = chestCount + 1
            local parent = safeProp(child, "Parent")
            local pos = getObjectPosition(child)
            qlog(string_format("    -> CHEST! parent=%s pos=%s", tostring(parent ~= nil), tostring(pos)))
        end
    end
    qlog("Total workspace children: " .. tostring(wsCount))
    qlog("Chests found in workspace: " .. tostring(chestCount))

    -- Also scan one level deep — check children of each workspace child
    qlog("Scanning 1 level deep:")
    for _, child in ipairs(safeChildren(Workspace)) do
        local childName = safeProp(child, "Name") or ""
        for _, grandchild in ipairs(safeChildren(child)) do
            local name = safeProp(grandchild, "Name") or ""
            if name == "Chest1" or name == "Chest2" or name == "Chest3" or string_find(string_lower(name), "chest", 1, true) then
                local pos = getObjectPosition(grandchild)
                qlog(string_format("  FOUND %s inside %s pos=%s", name, childName, tostring(pos)))
                chestCount = chestCount + 1
            end
        end
    end
    qlog("Total chests (incl. nested): " .. tostring(chestCount))
    qlog("Chests in NearbyChests: " .. tostring(#NearbyChests))
    for i, chest in ipairs(NearbyChests) do
        qlog(string_format("  [%d] %s dist=%.0f pos=%s", i, chest.Name, chest.Distance or -1, tostring(chest.Position)))
    end
    qlog("=== END CHEST DEBUG ===")
    notifyUser("Chest debug logged to F9 console (" .. tostring(#NearbyChests) .. " chests found)", 5)
end)

-- Visuals tab
local visualsTab = UILib:Tab("Visuals")
local themeSection = visualsTab:Section("Theme")
themeSection:Toggle("Watermark", Config.Watermark, function(v)
    Config.Watermark = v == true
    pcall(function() UILib:SetWatermarkEnabled(Config.Watermark) end)
end)

-- Misc / Debug tab
local miscTab = UILib:Tab("Misc")
local miscSection = miscTab:Section("Auto Features")
miscSection:Toggle("Auto Buso (Enhancement)", Config.AutoBuso, function(v)
    Config.AutoBuso = v == true
    if Config.AutoBuso then notifyUser("Auto Buso ON — will enable Enhancement Haki during combat", 3) end
end)
miscSection:Toggle("Server Hop", Config.ServerHop, function(v)
    Config.ServerHop = v == true
    if Config.ServerHop then notifyUser("Server Hop ON — will hop when no chests available", 3) end
end)
miscSection:Slider("Hop delay", Config.ServerHopDelay, 5, 10, 300, " sec", function(v)
    Config.ServerHopDelay = v
end)
miscSection:Toggle("Fruit ESP", Config.FruitNametags, function(v)
    Config.FruitNametags = v == true
end)
miscSection:Slider("Fruit range", Config.FruitRange, 50, 100, 5000, " studs", function(v)
    Config.FruitRange = v
end)

local miscDebugSection = miscTab:Section("Debug")
miscDebugSection:Button("Print player pos", function()
    local root = getLocalRoot()
    local pos = root and safeProp(root, "Position")
    if pos then
        local text = "Player position: " ..
            tostring(math_floor(pos.X)) .. ", " ..
            tostring(math_floor(pos.Y)) .. ", " ..
            tostring(math_floor(pos.Z))
        print(text)
        if type(setclipboard) == "function" then
            pcall(setclipboard, text)
        end
        notifyUser(text, 6)
    else
        notifyUser("Player position unavailable", 4)
    end
end)
miscDebugSection:Button("Print route", function()
    updateQuestRoute()
    local text = "Route: level " .. tostring(LevelState.CurrentLevel) ..
        " -> " .. tostring(LevelState.RouteEnemy) ..
        " @ " .. tostring(LevelState.RouteIsland) ..
        " via " .. tostring(LevelState.RouteQuestGiver) ..
        " (option " .. tostring(LevelState.RouteOption) .. ")"
    print(text)
    notifyUser(text, 6)
end)
miscDebugSection:Button("Rescan NPCs", function()
    scanNPCs()
    notifyUser("NPCs scanned: " .. tostring(#NearbyNPCs), 3)
end)
miscDebugSection:Button("Scan fruits", function()
    scanFruits()
    local n = #NearbyFruits
    local msg = "Fruits found: " .. tostring(n)
    if n > 0 then
        local lines = { msg }
        for i, fruit in ipairs(NearbyFruits) do
            local pos = fruit.Position
            table.insert(lines, string_format(
                "  %d. %s (raw=%s, rarity=%s) dist=%.0f pos=(%.0f,%.0f,%.0f)",
                i, tostring(fruit.Name), tostring(fruit.RawName),
                tostring(fruit.Rarity), fruit.Distance,
                pos.X, pos.Y, pos.Z
            ))
        end
        msg = table_concat(lines, "\n")
    end
    print(msg)
    if type(setclipboard) == "function" then pcall(setclipboard, msg) end
    notifyUser("Fruits: " .. tostring(n) .. " (see console/copied)", 5)
end)
miscDebugSection:Button("Deep scan workspace", function()
    -- Dumps all top-level Workspace children + looks for any folder/object
    -- whose name might contain "fruit" or "spawner", so we can find the
    -- real path if scanFruits returns 0.
    local lines = { "=== Workspace top-level children ===" }
    for _, child in ipairs(safeChildren(Workspace)) do
        local name = safeProp(child, "Name") or "?"
        local cn   = safeProp(child, "ClassName") or "?"
        local pos  = getObjectPosition(child)
        local posStr = "(no pos)"
        if pos then posStr = string_format("(%.0f,%.0f,%.0f)", pos.X, pos.Y, pos.Z) end
        local lower = string_lower(name)
        local marker = ""
        if string_find(lower, "fruit", 1, true) or
           string_find(lower, "spawner", 1, true) or
           string_find(lower, "devil", 1, true) then
            marker = "  <== CANDIDATE"
        end
        table.insert(lines, string_format("  %s [%s] %s%s", name, cn, posStr, marker))
    end
    -- Also check inside DevilFruitSpawner if it exists
    local spawner = safeFind(Workspace, "DevilFruitSpawner")
    if spawner then
        table.insert(lines, "=== Workspace.DevilFruitSpawner children ===")
        for _, child in ipairs(safeChildren(spawner)) do
            local name = safeProp(child, "Name") or "?"
            local cn   = safeProp(child, "ClassName") or "?"
            local n = 0
            for _ in pairs(safeChildren(child)) do n = n + 1 end
            table.insert(lines, string_format("  %s [%s] children=%d", name, cn, n))
        end
    else
        table.insert(lines, "(no Workspace.DevilFruitSpawner)")
    end
    -- Also check Workspace.Fruits
    local fruitsFolder = safeFind(Workspace, "Fruits")
    if fruitsFolder then
        table.insert(lines, "=== Workspace.Fruits children ===")
        for _, child in ipairs(safeChildren(fruitsFolder)) do
            local name = safeProp(child, "Name") or "?"
            local cn   = safeProp(child, "ClassName") or "?"
            local hasHandle = safeFind(child, "Handle") and "yes" or "no"
            table.insert(lines, string_format("  %s [%s] Handle=%s", name, cn, hasHandle))
        end
    else
        table.insert(lines, "(no Workspace.Fruits folder)")
    end
    local msg = table_concat(lines, "\n")
    print(msg)
    if type(setclipboard) == "function" then pcall(setclipboard, msg) end
    notifyUser("Workspace dump printed + copied", 5)
end)
miscDebugSection:Button("Unload", function()
    Running = false
end)

-- Settings tab (UILib built-in)
pcall(function() UILib:CreateSettingsTab("Settings") end)

-- Final init
notifyUser("Blox Fruits loaded. Press F1 for menu.", 6)
qlog("=== Companion initialized ===")
local _vpX, _vpY = getViewportSize()
qlog(string_format("Viewport: %d x %d", _vpX, _vpY))
qlog(string_format("Mouse variant: %s", tostring(_mousemoveabs_variant or "(undetected yet)")))
qlog("Level: " .. tostring(LevelState.CurrentLevel) ..
     " | Route enemy: " .. tostring(LevelState.RouteEnemy) ..
     " | Island: " .. tostring(LevelState.RouteIsland))

-- ============================================================================
-- 22. MAIN LOOPS
-- ============================================================================

-- Background NPC scanner — runs at 1Hz (was 10Hz — caused massive lag)
-- We don't need fresh NPC data 10x per second; 1x per second is plenty for
-- ESP and target selection. The fight loop still re-scans on demand.
task_spawn(function()
    -- Initial scan immediately
    task_wait(0.1)
    scanNPCs()
    while Running do
        task_wait(1.0)  -- 1 Hz — 10x less work than before
        pcall(scanNPCs)
    end
end)

-- Main frame loop
local _lastGuiCacheRefresh = 0
local _lastChestRangeDraw  = 0
local _lastHudUpdate       = 0
while Running do
    pcall(function() UILib:Step() end)

    -- Refresh GUI cache every 0.5s (cheap — walks the chain once and caches
    -- all references so getQuestTrackerText/isQuestTrackerVisible become
    -- a single safeProp call instead of 5-7 safeFind calls per frame).
    if os_clock() - _lastGuiCacheRefresh > 0.5 then
        pcall(refreshGuiCache)
        _lastGuiCacheRefresh = os_clock()
    end

    renderNpcOverlays()

    -- Render chest range visualization — throttled to 10 Hz (was 60 fps).
    -- Circle updates don't need frame-perfect smoothness; the player moves
    -- slowly relative to the circle radius.
    if Config.ChestVisualRange then
        if os_clock() - _lastChestRangeDraw > 0.1 then
            _lastChestRangeDraw = os_clock()
            local root = getLocalRoot()
            local rootPos = root and safeProp(root, "Position")
            if rootPos then
                for i = 1, RANGE_SEGMENTS do
                    local angle1 = ((i - 1) / RANGE_SEGMENTS) * math_pi * 2
                    local angle2 = (i / RANGE_SEGMENTS) * math_pi * 2
                    local p1 = rootPos + Vector3_new(math_cos(angle1) * Config.ChestRange, 0, math_sin(angle1) * Config.ChestRange)
                    local p2 = rootPos + Vector3_new(math_cos(angle2) * Config.ChestRange, 0, math_sin(angle2) * Config.ChestRange)
                    local ok1, s1, on1 = pcall(WorldToScreen, p1)
                    local ok2, s2, on2 = pcall(WorldToScreen, p2)
                    if ok1 and s1 and on1 and ok2 and s2 and on2 then
                        rangeCircleLines[i].From = Vector2_new(math_floor(s1.X + 0.5), math_floor(s1.Y + 0.5))
                        rangeCircleLines[i].To   = Vector2_new(math_floor(s2.X + 0.5), math_floor(s2.Y + 0.5))
                        rangeCircleLines[i].Visible = true
                    else
                        rangeCircleLines[i].Visible = false
                    end
                end
            end
        end
    else
        for _, l in ipairs(rangeCircleLines) do l.Visible = false end
    end

    -- (Chest ESP is now handled inside chestProcessStep)

    -- Update Quest Progress HUD — throttled to 5 Hz (was every frame).
    -- HUD text changes at most when kill count changes (every few seconds),
    -- so 5 Hz is plenty for visual feedback.
    if os_clock() - _lastHudUpdate > 0.2 then
        _lastHudUpdate = os_clock()
        local hudText = getQuestTrackerText()
        if hudText and #hudText > 5 then
            -- Line 1: Quest objective (yellow)
            QuestHUD.Text = "[Quest] " .. hudText
            QuestHUD.Visible = true

            -- Line 2: Route info (blue)
            QuestHUD2.Text = "Route: " .. tostring(LevelState.RouteEnemy) ..
                             " @ " .. tostring(LevelState.RouteIsland) ..
                             " via " .. tostring(LevelState.RouteQuestGiver)
            QuestHUD2.Visible = true

            -- Line 3: Status (green)
            local statusText = "Status: " .. tostring(LevelState.QuestPhase)
            if LevelState.Target and LevelState.Target ~= "none" then
                statusText = statusText .. " | Target: " .. tostring(LevelState.Target)
            end
            if LevelState.QuestAccepted then
                local killCurrent, killTotal = string_match(tostring(hudText), "%((%d+)%s*/%s*(%d+)%)")
                if killCurrent and killTotal then
                    statusText = statusText .. " | Kills: " .. killCurrent .. "/" .. killTotal
                end
            end
            QuestHUD3.Text = statusText
            QuestHUD3.Visible = true
        else
            QuestHUD.Text = ""
            QuestHUD.Visible = false
            QuestHUD2.Text = ""
            QuestHUD2.Visible = false
            QuestHUD3.Text = ""
            QuestHUD3.Visible = false
        end
    end

    levelProcessStep()
    chestProcessStep()
    releaseClickIfNeeded()
    task_wait()
end

cleanup()

-- ============================================================================
-- End of script
-- ============================================================================
