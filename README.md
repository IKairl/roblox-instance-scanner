
--[[
    INSTANCE SCANNER v0.4
    Explorer-style Instance Browser

    Features:
    - Recursive instance scanning
    - Expandable hierarchy
    - Recursive descendant search
    - Automatic scrolling canvas sizing
    - Refresh button
    - Instance property inspection
    - Manual-copy information window
    - Periodic yielding during large scans
    - Minimize / restore control
    - Resizable window

    Read-only inspection tool.
]]

--==================================================
-- SERVICES
--==================================================

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

--==================================================
-- CONFIGURATION
--==================================================

local CONFIG = {
    ScanRoot = workspace,
    MaxDepth = 30,

    WindowWidth = 650,
    WindowHeight = 430,
    MinWindowWidth = 520,
    MinWindowHeight = 320,
    MaxWindowWidth = 1100,
    MaxWindowHeight = 850,

    Background = Color3.fromRGB(25, 27, 33),
    Secondary = Color3.fromRGB(35, 38, 46),
    Accent = Color3.fromRGB(65, 130, 230),
    Text = Color3.fromRGB(235, 235, 235),
    Muted = Color3.fromRGB(155, 160, 170),

    RowHeight = 25,
    ScanYieldInterval = 150,
}

--==================================================
-- REMOVE PREVIOUS GUI
--==================================================

pcall(function()
    local old = CoreGui:FindFirstChild("InstanceScanner")
    if old then old:Destroy() end
end)

pcall(function()
    local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
    if playerGui then
        local old = playerGui:FindFirstChild("InstanceScanner")
        if old then old:Destroy() end
    end
end)

--==================================================
-- STATE
--==================================================

local ScanData = {}
local SelectedRecord = nil
local Expanded = {}
local RowObjects = {}

local TotalScanned = 0
local IsScanning = false
local RenderOrder = 0

local IsMinimized = false
local ExpandedSize = UDim2.fromOffset(CONFIG.WindowWidth, CONFIG.WindowHeight)

--==================================================
-- GUI CREATION
--==================================================

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "InstanceScanner"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
