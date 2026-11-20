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
    - Touch / mobile friendly layout (v0.4 mobile patch)

    Read-only inspection tool.
]]

--==================================================
-- SERVICES
--==================================================

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")

if not game:IsLoaded() then
    game.Loaded:Wait()
end

local LocalPlayer = Players.LocalPlayer

--==================================================
-- MOBILE / SCREEN HELPERS
--==================================================

local IsTouch = UserInputService.TouchEnabled

-- Optional executor clipboard function (used for the COPY button if present).
local ClipboardFn = setclipboard or toclipboard

local function GetViewportSize()
    local camera = workspace.CurrentCamera
    local size = camera and camera.ViewportSize or Vector2.new(0, 0)

    -- Fallback if the camera isn't ready yet.
    if size.X < 100 or size.Y < 100 then
        size = Vector2.new(800, 400)
    end

    return size
end

local Viewport = GetViewportSize()
local FIT_W = Viewport.X - 24 -- usable width for the window
local FIT_H = Viewport.Y - 40 -- usable height for the window

-- Slightly larger text on touch screens.
local function TS(size)
    return IsTouch and (size + 2) or size
end

-- Bigger tap targets on touch screens.
local COPY_BUTTON_HEIGHT = IsTouch and 34 or 28
local RESIZE_GRIP = IsTouch and 32 or 20

--==================================================
-- CONFIGURATION
--==================================================

local CONFIG = {
    ScanRoot = workspace,
    MaxDepth = 30,

    -- Sizes are clamped to the screen so the window can't spawn off-screen on phones.
    WindowWidth = math.floor(math.min(650, FIT_W)),
    WindowHeight = math.floor(math.min(430, FIT_H)),
    MinWindowWidth = math.floor(math.min(IsTouch and 420 or 520, FIT_W)),
    MinWindowHeight = math.floor(math.min(IsTouch and 250 or 320, FIT_H)),
    MaxWindowWidth = 1100,
    MaxWindowHeight = 850,

    Background = Color3.fromRGB(25, 27, 33),
    Secondary = Color3.fromRGB(35, 38, 46),
    Accent = Color3.fromRGB(65, 130, 230),
    Text = Color3.fromRGB(235, 235, 235),
    Muted = Color3.fromRGB(155, 160, 170),

    RowHeight = IsTouch and 32 or 25,
    ScanYieldInterval = IsTouch and 75 or 150,
}

--==================================================
-- REMOVE PREVIOUS GUI
--==================================================

local function DestroyOld(container)
    pcall(function()
        local old = container:FindFirstChild("InstanceScanner")
        if old then old:Destroy() end
    end)
end

DestroyOld(CoreGui)

if gethui then
    pcall(function()
        DestroyOld(gethui())
    end)
end

pcall(function()
    DestroyOld(LocalPlayer:FindFirstChild("PlayerGui"))
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
ScreenGui.IgnoreGuiInset = true -- keeps the top bar / notch from shifting the window
ScreenGui.DisplayOrder = 999

-- Try the executor's hidden UI container first (best on mobile executors),
-- then CoreGui, then PlayerGui.
local function ParentGui(gui)
    local getters = {}

    if gethui then
        table.insert(getters, function()
            return gethui()
        end)
    end

    table.insert(getters, function()
        return CoreGui
    end)

    table.insert(getters, function()
        return LocalPlayer:WaitForChild("PlayerGui")
    end)

    for _, getContainer in ipairs(getters) do
        local ok = pcall(function()
            gui.Parent = getContainer()
        end)

        if ok and gui.Parent then
            return
        end
    end
end

ParentGui(ScreenGui)

local Main = Instance.new("Frame")
Main.Name = "MainWindow"
Main.Size = UDim2.fromOffset(
    CONFIG.WindowWidth,
    CONFIG.WindowHeight
)
-- Centered on the actual screen (top-left anchored so minimize keeps the corner in place).
Main.Position = UDim2.fromOffset(
    math.floor((Viewport.X - CONFIG.WindowWidth) / 2),
    math.floor((Viewport.Y - CONFIG.WindowHeight) / 2)
)
Main.BackgroundColor3 = CONFIG.Background
Main.BorderSizePixel = 0
Main.Active = true -- stops touches from also rotating the camera / moving the character
Main.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 9)
MainCorner.Parent = Main

--==================================================
-- HEADER
--==================================================

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 42)
Header.BackgroundColor3 = CONFIG.Secondary
Header.BorderSizePixel = 0
Header.Parent = Main

local HeaderTitle = Instance.new("TextLabel")
HeaderTitle.Size = UDim2.new(1, IsTouch and -66 or -58, 1, 0)
HeaderTitle.Position = UDim2.fromOffset(14, 0)
HeaderTitle.BackgroundTransparency = 1
HeaderTitle.Text = "INSTANCE SCANNER  |  v0.4"
HeaderTitle.TextColor3 = CONFIG.Text
HeaderTitle.Font = Enum.Font.GothamBold
HeaderTitle.TextSize = TS(13)
HeaderTitle.TextXAlignment = Enum.TextXAlignment.Left
HeaderTitle.Parent = Header

-- Minimize / restore button
local MinimizeButton = Instance.new("TextButton")
MinimizeButton.Name = "MinimizeButton"
MinimizeButton.Size = IsTouch and UDim2.fromOffset(38, 34) or UDim2.fromOffset(28, 26)
MinimizeButton.Position = IsTouch and UDim2.new(1, -46, 0, 4) or UDim2.new(1, -38, 0, 8)
MinimizeButton.BackgroundColor3 = CONFIG.Background
MinimizeButton.Text = "−"
MinimizeButton.TextColor3 = CONFIG.Text
MinimizeButton.Font = Enum.Font.GothamBold
MinimizeButton.TextSize = 18
MinimizeButton.BorderSizePixel = 0
MinimizeButton.Parent = Header

local MinimizeCorner = Instance.new("UICorner")
MinimizeCorner.CornerRadius = UDim.new(0, 5)
MinimizeCorner.Parent = MinimizeButton

--==================================================
-- SEARCH BAR
--==================================================

local SearchBox = Instance.new("TextBox")
SearchBox.Size = UDim2.new(1, -116, 0, 32)
SearchBox.Position = UDim2.fromOffset(14, 53)
SearchBox.BackgroundColor3 = CONFIG.Secondary
SearchBox.TextColor3 = CONFIG.Text
SearchBox.PlaceholderText = "Search instances..."
SearchBox.PlaceholderColor3 = CONFIG.Muted
SearchBox.Font = Enum.Font.Gotham
SearchBox.TextSize = TS(12)
SearchBox.ClearTextOnFocus = false
SearchBox.TextXAlignment = Enum.TextXAlignment.Left
SearchBox.Parent = Main

local SearchPadding = Instance.new("UIPadding")
SearchPadding.PaddingLeft = UDim.new(0, 9)
SearchPadding.Parent = SearchBox

local SearchCorner = Instance.new("UICorner")
SearchCorner.CornerRadius = UDim.new(0, 5)
SearchCorner.Parent = SearchBox

--==================================================
-- REFRESH BUTTON
--==================================================

local RefreshButton = Instance.new("TextButton")
RefreshButton.Size = UDim2.fromOffset(88, 32)
RefreshButton.Position = UDim2.new(1, -102, 0, 53)
RefreshButton.BackgroundColor3 = CONFIG.Accent
RefreshButton.Text = "REFRESH"
RefreshButton.TextColor3 = CONFIG.Text
RefreshButton.Font = Enum.Font.GothamBold
RefreshButton.TextSize = TS(10)
RefreshButton.BorderSizePixel = 0
RefreshButton.Parent = Main

local RefreshCorner = Instance.new("UICorner")
RefreshCorner.CornerRadius = UDim.new(0, 5)
RefreshCorner.Parent = RefreshButton

--==================================================
-- EXPLORER PANEL
--==================================================

local Explorer = Instance.new("ScrollingFrame")
Explorer.Name = "Explorer"
Explorer.Size = UDim2.new(0.52, -18, 1, -150)
Explorer.Position = UDim2.fromOffset(14, 95)
Explorer.BackgroundColor3 = CONFIG.Secondary
Explorer.BorderSizePixel = 0
Explorer.ScrollBarThickness = IsTouch and 8 or 6
Explorer.ScrollBarImageColor3 = CONFIG.Accent
Explorer.ScrollingDirection = Enum.ScrollingDirection.Y
Explorer.AutomaticCanvasSize = Enum.AutomaticSize.Y
Explorer.CanvasSize = UDim2.new(0, 0, 0, 0)
Explorer.ClipsDescendants = true
Explorer.Parent = Main

local ExplorerCorner = Instance.new("UICorner")
ExplorerCorner.CornerRadius = UDim.new(0, 5)
ExplorerCorner.Parent = Explorer

local ExplorerPadding = Instance.new("UIPadding")
ExplorerPadding.PaddingTop = UDim.new(0, 5)
ExplorerPadding.PaddingBottom = UDim.new(0, 5)
ExplorerPadding.PaddingLeft = UDim.new(0, 4)
ExplorerPadding.PaddingRight = UDim.new(0, 4)
ExplorerPadding.Parent = Explorer

local ExplorerLayout = Instance.new("UIListLayout")
ExplorerLayout.Padding = UDim.new(0, 2)
ExplorerLayout.SortOrder = Enum.SortOrder.LayoutOrder
ExplorerLayout.Parent = Explorer

ExplorerLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    Explorer.CanvasSize = UDim2.new(
        0, 0, 0,
        ExplorerLayout.AbsoluteContentSize.Y + 10
    )
end)

--==================================================
-- PROPERTIES PANEL
--==================================================

local Properties = Instance.new("Frame")
Properties.Name = "Properties"
Properties.Size = UDim2.new(0.48, -24, 1, -150)
Properties.Position = UDim2.new(0.52, 4, 0, 95)
Properties.BackgroundColor3 = CONFIG.Secondary
Properties.BorderSizePixel = 0
Properties.Parent = Main

local PropertiesCorner = Instance.new("UICorner")
PropertiesCorner.CornerRadius = UDim.new(0, 5)
PropertiesCorner.Parent = Properties

local PropertyTitle = Instance.new("TextLabel")
PropertyTitle.Size = UDim2.new(1, -16, 0, 32)
PropertyTitle.Position = UDim2.fromOffset(8, 0)
PropertyTitle.BackgroundTransparency = 1
PropertyTitle.Text = "INSTANCE DETAILS"
PropertyTitle.TextColor3 = CONFIG.Text
PropertyTitle.Font = Enum.Font.GothamBold
PropertyTitle.TextSize = TS(12)
PropertyTitle.TextXAlignment = Enum.TextXAlignment.Left
PropertyTitle.Parent = Properties

local PropertyInfo = Instance.new("TextLabel")
PropertyInfo.Size = UDim2.new(1, -16, 1, -(COPY_BUTTON_HEIGHT + 56))
PropertyInfo.Position = UDim2.fromOffset(8, 36)
PropertyInfo.BackgroundTransparency = 1
PropertyInfo.Text = "Select an instance to inspect."
PropertyInfo.TextColor3 = CONFIG.Muted
PropertyInfo.Font = Enum.Font.Code
PropertyInfo.TextSize = TS(11)
PropertyInfo.TextWrapped = true
PropertyInfo.TextYAlignment = Enum.TextYAlignment.Top
PropertyInfo.TextXAlignment = Enum.TextXAlignment.Left
PropertyInfo.Parent = Properties

local CopyButton = Instance.new("TextButton")
CopyButton.Size = UDim2.new(1, -16, 0, COPY_BUTTON_HEIGHT)
CopyButton.Position = UDim2.new(0, 8, 1, -(COPY_BUTTON_HEIGHT + 8))
CopyButton.BackgroundColor3 = CONFIG.Accent
CopyButton.Text = "VIEW / COPY INSTANCE INFO"
CopyButton.TextColor3 = CONFIG.Text
CopyButton.Font = Enum.Font.GothamBold
CopyButton.TextSize = TS(10)
CopyButton.BorderSizePixel = 0
CopyButton.Parent = Properties

local CopyCorner = Instance.new("UICorner")
CopyCorner.CornerRadius = UDim.new(0, 5)
CopyCorner.Parent = CopyButton

--==================================================
-- MANUAL COPY WINDOW
--==================================================

local CopyWindow = Instance.new("Frame")
CopyWindow.Name = "CopyWindow"
CopyWindow.AnchorPoint = Vector2.new(0.5, 0.5)
CopyWindow.Size = UDim2.fromScale(0.94, 0.88) -- scales with the window, capped below
CopyWindow.Position = UDim2.fromScale(0.5, 0.5)
CopyWindow.Active = true
CopyWindow.BackgroundColor3 = CONFIG.Background
CopyWindow.BorderSizePixel = 0
CopyWindow.Visible = false
CopyWindow.ZIndex = 5
CopyWindow.Parent = Main

local CopyWindowCorner = Instance.new("UICorner")
CopyWindowCorner.CornerRadius = UDim.new(0, 8)
CopyWindowCorner.Parent = CopyWindow

local CopyWindowLimit = Instance.new("UISizeConstraint")
CopyWindowLimit.MaxSize = Vector2.new(430, 270)
CopyWindowLimit.Parent = CopyWindow

local CopyTitle = Instance.new("TextLabel")
CopyTitle.Size = UDim2.new(1, -50, 0, 35)
CopyTitle.Position = UDim2.fromOffset(12, 0)
CopyTitle.BackgroundTransparency = 1
CopyTitle.Text = "INSTANCE INFORMATION"
CopyTitle.TextColor3 = CONFIG.Text
CopyTitle.Font = Enum.Font.GothamBold
CopyTitle.TextSize = TS(12)
CopyTitle.TextXAlignment = Enum.TextXAlignment.Left
CopyTitle.ZIndex = 6
CopyTitle.Parent = CopyWindow

local CloseCopy = Instance.new("TextButton")
CloseCopy.Size = IsTouch and UDim2.fromOffset(38, 30) or UDim2.fromOffset(30, 27)
CloseCopy.Position = IsTouch and UDim2.new(1, -46, 0, 3) or UDim2.new(1, -38, 0, 4)
CloseCopy.BackgroundColor3 = CONFIG.Secondary
CloseCopy.Text = "X"
CloseCopy.TextColor3 = CONFIG.Text
CloseCopy.Font = Enum.Font.GothamBold
CloseCopy.TextSize = TS(12)
CloseCopy.BorderSizePixel = 0
CloseCopy.ZIndex = 6
CloseCopy.Parent = CopyWindow

local CopyTextBox = Instance.new("TextBox")
CopyTextBox.Size = UDim2.new(1, -24, 1, -83)
CopyTextBox.Position = UDim2.fromOffset(12, 40)
CopyTextBox.BackgroundColor3 = CONFIG.Secondary
CopyTextBox.TextColor3 = CONFIG.Text
CopyTextBox.PlaceholderText = "Instance information will appear here."
CopyTextBox.Font = Enum.Font.Code
CopyTextBox.TextSize = TS(11)
CopyTextBox.TextXAlignment = Enum.TextXAlignment.Left
CopyTextBox.TextYAlignment = Enum.TextYAlignment.Top
CopyTextBox.TextWrapped = true
CopyTextBox.MultiLine = true
CopyTextBox.ClearTextOnFocus = false
CopyTextBox.TextEditable = true
CopyTextBox.ZIndex = 6
CopyTextBox.Parent = CopyWindow

local CopyTextCorner = Instance.new("UICorner")
CopyTextCorner.CornerRadius = UDim.new(0, 5)
CopyTextCorner.Parent = CopyTextBox

local CopyInstructions = Instance.new("TextLabel")
CopyInstructions.Size = UDim2.new(1, ClipboardFn and -146 or -24, 0, 30)
CopyInstructions.Position = UDim2.new(0, 12, 1, -36)
CopyInstructions.BackgroundTransparency = 1
if IsTouch then
    CopyInstructions.Text = ClipboardFn
        and "Tap COPY, or tap and hold the text to select it."
        or "Tap and hold the text to select it, then copy."
else
    CopyInstructions.Text = ClipboardFn
        and "Click COPY, or click the text and press Ctrl + A, then Ctrl + C."
        or "Click the text, press Ctrl + A, then Ctrl + C."
end
CopyInstructions.TextColor3 = CONFIG.Muted
CopyInstructions.Font = Enum.Font.Gotham
CopyInstructions.TextSize = TS(10)
CopyInstructions.TextWrapped = true
CopyInstructions.TextXAlignment = Enum.TextXAlignment.Left
CopyInstructions.ZIndex = 6
CopyInstructions.Parent = CopyWindow

-- One-tap copy button, only created if the executor provides a clipboard function.
local CopyClipboard

if ClipboardFn then
    CopyClipboard = Instance.new("TextButton")
    CopyClipboard.Size = UDim2.fromOffset(110, 30)
    CopyClipboard.Position = UDim2.new(1, -122, 1, -38)
    CopyClipboard.BackgroundColor3 = CONFIG.Accent
    CopyClipboard.Text = "COPY"
    CopyClipboard.TextColor3 = CONFIG.Text
    CopyClipboard.Font = Enum.Font.GothamBold
    CopyClipboard.TextSize = TS(10)
    CopyClipboard.BorderSizePixel = 0
    CopyClipboard.ZIndex = 6
    CopyClipboard.Parent = CopyWindow

    local CopyClipboardCorner = Instance.new("UICorner")
    CopyClipboardCorner.CornerRadius = UDim.new(0, 5)
    CopyClipboardCorner.Parent = CopyClipboard
end

--==================================================
-- FOOTER
--==================================================

local Footer = Instance.new("TextLabel")
Footer.Size = UDim2.new(1, -(RESIZE_GRIP + 24), 0, 20)
Footer.Position = UDim2.new(0, 14, 1, -25)
Footer.BackgroundTransparency = 1
Footer.Text = "Waiting for scan..."
Footer.TextColor3 = CONFIG.Muted
Footer.Font = Enum.Font.Gotham
Footer.TextSize = TS(10)
Footer.TextXAlignment = Enum.TextXAlignment.Left
Footer.Parent = Main

-- Bottom-right resize handle
local ResizeGrip = Instance.new("TextButton")
ResizeGrip.Name = "ResizeGrip"
ResizeGrip.Size = UDim2.fromOffset(RESIZE_GRIP, RESIZE_GRIP)
ResizeGrip.Position = UDim2.new(1, -RESIZE_GRIP, 1, -RESIZE_GRIP)
ResizeGrip.BackgroundColor3 = CONFIG.Secondary
ResizeGrip.BackgroundTransparency = 0.1
ResizeGrip.Text = "◢"
ResizeGrip.TextColor3 = CONFIG.Muted
ResizeGrip.Font = Enum.Font.GothamBold
ResizeGrip.TextSize = TS(12)
ResizeGrip.BorderSizePixel = 0
ResizeGrip.AutoButtonColor = true
ResizeGrip.Parent = Main

local ResizeGripCorner = Instance.new("UICorner")
ResizeGripCorner.CornerRadius = UDim.new(0, 4)
ResizeGripCorner.Parent = ResizeGrip

--==================================================
-- INSTANCE SCANNER
--==================================================

local function ScanRecursive(parent, depth)

    local results = {}

    if depth > CONFIG.MaxDepth then
        return results
    end

    local success, children = pcall(function()
        return parent:GetChildren()
    end)

    if not success then
        return results
    end

    for _, object in ipairs(children) do

        TotalScanned += 1

        local fullName = object.Name

        pcall(function()
            fullName = object:GetFullName()
        end)

        local record = {
            Instance = object,
            Name = object.Name,
            ClassName = object.ClassName,
            FullName = fullName,
            Depth = depth,
            Children = {},
        }

        record.Children = ScanRecursive(
            object,
            depth + 1
        )

        table.insert(results, record)

        if TotalScanned % CONFIG.ScanYieldInterval == 0 then
            task.wait()
        end

    end

    return results
end

--==================================================
-- PROPERTY INSPECTOR
--==================================================

local function ShowProperties(record)

    SelectedRecord = record

    PropertyInfo.Text =
        "Name: " .. record.Name ..
        "\n\nClass: " .. record.ClassName ..
        "\n\nPath: " .. record.FullName ..
        "\n\nChildren: " .. tostring(#record.Children) ..
        "\n\nDepth: " .. tostring(record.Depth)

end

--==================================================
-- SEARCH HELPERS
--==================================================

local function ContainsQuery(name, query)

    if query == "" then
        return true
    end

    return string.find(
        string.lower(name),
        string.lower(query),
        1,
        true
    ) ~= nil

end

local function HasMatchingDescendant(record, query)

    if ContainsQuery(record.Name, query) then
        return true
    end

    for _, child in ipairs(record.Children) do

        if HasMatchingDescendant(child, query) then
            return true
        end

    end

    return false

end

--==================================================
-- EXPLORER RENDERER
--==================================================

local function ClearExplorer()

    for _, object in ipairs(RowObjects) do
        object:Destroy()
    end

    table.clear(RowObjects)
    RenderOrder = 0

end

local function RenderRecords(records, depth, query)

    for _, record in ipairs(records) do

        local matches = ContainsQuery(record.Name, query)
        local descendantMatches = false

        if query ~= "" and not matches then
            descendantMatches = HasMatchingDescendant(
                record,
                query
            )
        end

        if matches or descendantMatches then

            RenderOrder += 1

            local button = Instance.new("TextButton")
            button.Name = "InstanceRow"
            button.LayoutOrder = RenderOrder
            button.Size = UDim2.new(1, -2, 0, CONFIG.RowHeight)
            button.BackgroundColor3 = CONFIG.Background
            button.BackgroundTransparency = 0.2
            button.BorderSizePixel = 0

            local indent = math.min(depth - 1, 12)

            local hasChildren = #record.Children > 0
            local isExpanded = Expanded[record.FullName]

            local arrow = "   "

            if hasChildren then
                arrow = (query ~= "" or isExpanded)
                    and "▼ " or "▶ "
            end

            button.Text =
                string.rep("   ", indent) ..
                arrow ..
                record.Name ..
                "  [" .. record.ClassName .. "]"

            button.TextColor3 = CONFIG.Text
            button.Font = Enum.Font.Gotham
            button.TextSize = TS(10)
            button.TextXAlignment = Enum.TextXAlignment.Left
            button.TextTruncate = Enum.TextTruncate.AtEnd
            button.Parent = Explorer

            if SelectedRecord == record then
                button.BackgroundColor3 = CONFIG.Accent
                button.BackgroundTransparency = 0.15
            end

            button.MouseButton1Click:Connect(function()

                ShowProperties(record)

                if hasChildren and query == "" then

                    Expanded[record.FullName] =
                        not Expanded[record.FullName]

                end

                ClearExplorer()

                RenderRecords(
                    ScanData,
                    1,
                    SearchBox.Text
                )

            end)

            table.insert(RowObjects, button)

            if query ~= "" then

                if descendantMatches then
                    RenderRecords(
                        record.Children,
                        depth + 1,
                        query
                    )
                end

            elseif isExpanded then

                RenderRecords(
                    record.Children,
                    depth + 1,
                    ""
                )

            end

        end

    end

end

--==================================================
-- SCAN FUNCTION
--==================================================

local function PerformScan()

    if IsScanning then
        return
    end

    IsScanning = true

    Footer.Text = "Scanning accessible instances..."

    RefreshButton.Text = "SCANNING"
    RefreshButton.Active = false
    RefreshButton.AutoButtonColor = false

    table.clear(ScanData)
    table.clear(Expanded)

    SelectedRecord = nil
    PropertyInfo.Text = "Select an instance to inspect."

    TotalScanned = 0

    task.spawn(function()

        local success, result = pcall(function()

            return ScanRecursive(
                CONFIG.ScanRoot,
                1
            )

        end)

        if success then

            ScanData = result

            ClearExplorer()

            RenderRecords(
                ScanData,
                1,
                SearchBox.Text
            )

            Footer.Text =
                "Scan complete | " ..
                tostring(TotalScanned) ..
                " instances discovered"

        else

            Footer.Text = "Scan failed: " .. tostring(result)

        end

        IsScanning = false

        RefreshButton.Text = "REFRESH"
        RefreshButton.Active = true
        RefreshButton.AutoButtonColor = true

    end)

end

--==================================================
-- SEARCH EVENT
--==================================================

SearchBox:GetPropertyChangedSignal("Text"):Connect(function()

    ClearExplorer()

    RenderRecords(
        ScanData,
        1,
        SearchBox.Text
    )

end)

--==================================================
-- REFRESH EVENT
--==================================================

RefreshButton.MouseButton1Click:Connect(function()
    PerformScan()
end)

--==================================================
-- MANUAL COPY EVENT
--==================================================

CopyButton.MouseButton1Click:Connect(function()

    if not SelectedRecord then
        Footer.Text = "Select an instance first."
        return
    end

    local record = SelectedRecord

    local copyText =
        "Name: " .. record.Name ..
        "\nClass: " .. record.ClassName ..
        "\nPath: " .. record.FullName ..
        "\nChildren: " .. tostring(#record.Children) ..
        "\nDepth: " .. tostring(record.Depth)

    CopyTextBox.Text = copyText
    CopyWindow.Visible = true

    Footer.Text = "Information window opened."

    -- On phones, focusing the box instantly opens the keyboard over the window,
    -- so only auto-focus when there's a mouse/keyboard.
    if not IsTouch then
        task.defer(function()
            CopyTextBox:CaptureFocus()
            CopyTextBox.CursorPosition = 1
        end)
    end

end)

CloseCopy.MouseButton1Click:Connect(function()

    CopyWindow.Visible = false
    CopyTextBox:ReleaseFocus()

end)

if CopyClipboard then
    CopyClipboard.MouseButton1Click:Connect(function()
        local ok = pcall(ClipboardFn, CopyTextBox.Text)

        Footer.Text = ok
            and "Copied to clipboard."
            or "Copy failed - select the text manually."
    end)
end

--==================================================
-- MINIMIZE / RESTORE
--==================================================

local ContentObjects = {
    SearchBox,
    RefreshButton,
    Explorer,
    Properties,
    Footer,
    ResizeGrip,
}

MinimizeButton.MouseButton1Click:Connect(function()
    IsMinimized = not IsMinimized

    if IsMinimized then
        -- Preserve the most recent expanded dimensions before collapsing.
        ExpandedSize = Main.Size

        for _, object in ipairs(ContentObjects) do
            object.Visible = false
        end

        CopyWindow.Visible = false
        Main.Size = UDim2.fromOffset(ExpandedSize.X.Offset, 42)
        MinimizeButton.Text = "+"
        MinimizeButton.TextSize = 15
    else
        Main.Size = ExpandedSize

        for _, object in ipairs(ContentObjects) do
            object.Visible = true
        end

        MinimizeButton.Text = "−"
        MinimizeButton.TextSize = 18
    end
end)

--==================================================
-- DRAGGABLE WINDOW
--==================================================

-- Keeps the title bar reachable so the window can never be dragged fully off-screen.
local function ClampToScreen(x, y)
    local vp = GetViewportSize()
    local width = Main.AbsoluteSize.X

    x = math.clamp(x, -(width - 100), vp.X - 100)
    y = math.clamp(y, 0, vp.Y - 42)

    return x, y
end

-- Re-fit the window when the screen size changes (e.g. phone rotation).
local function FitToScreen()
    local vp = GetViewportSize()

    local width = math.floor(math.min(ExpandedSize.X.Offset, vp.X - 12))
    local height = math.floor(math.min(ExpandedSize.Y.Offset, vp.Y - 12))

    ExpandedSize = UDim2.fromOffset(width, height)
    Main.Size = IsMinimized and UDim2.fromOffset(width, 42) or ExpandedSize

    local x, y = ClampToScreen(Main.Position.X.Offset, Main.Position.Y.Offset)
    Main.Position = UDim2.fromOffset(x, y)
end

local dragging = false
local dragStart
local startPosition

local resizing = false
local resizeStart
local resizeStartSize

Header.InputBegan:Connect(function(input)

    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        dragStart = input.Position
        startPosition = Main.Position

    end

end)

UserInputService.InputChanged:Connect(function(input)

    if dragging then

        if input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch then

            local delta = input.Position - dragStart

            local x, y = ClampToScreen(
                startPosition.X.Offset + delta.X,
                startPosition.Y.Offset + delta.Y
            )

            Main.Position = UDim2.fromOffset(x, y)

        end

    end

end)

-- Begin resizing from the bottom-right grip.
ResizeGrip.InputBegan:Connect(function(input)
    if IsMinimized then
        return
    end

    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        resizing = true
        resizeStart = input.Position
        resizeStartSize = Main.Size
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not resizing then
        return
    end

    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then

        local delta = input.Position - resizeStart

        local vp = GetViewportSize()

        local newWidth = math.clamp(
            resizeStartSize.X.Offset + delta.X,
            CONFIG.MinWindowWidth,
            math.max(CONFIG.MinWindowWidth, math.min(CONFIG.MaxWindowWidth, vp.X - 12))
        )

        local newHeight = math.clamp(
            resizeStartSize.Y.Offset + delta.Y,
            CONFIG.MinWindowHeight,
            math.max(CONFIG.MinWindowHeight, math.min(CONFIG.MaxWindowHeight, vp.Y - 12))
        )

        Main.Size = UDim2.fromOffset(newWidth, newHeight)
        ExpandedSize = Main.Size
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        resizing = false
    end
end)

UserInputService.InputEnded:Connect(function(input)

    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = false

    end

end)

--==================================================
-- SCREEN SIZE CHANGES (ROTATION)
--==================================================

do
    local camera = workspace.CurrentCamera

    if camera then
        camera:GetPropertyChangedSignal("ViewportSize"):Connect(FitToScreen)
    end
end

--==================================================
-- INITIAL SCAN
--==================================================

PerformScan()

print("[Instance Scanner v0.4] Loaded successfully.")
