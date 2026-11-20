
--[[
    INSTANCE SCANNER v0.3
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

--==================================================
-- GUI CREATION
--==================================================

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "InstanceScanner"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local parentSuccess = pcall(function()
    ScreenGui.Parent = CoreGui
end)

if not parentSuccess or not ScreenGui.Parent then
    ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

local Main = Instance.new("Frame")
Main.Name = "MainWindow"
Main.Size = UDim2.fromOffset(
    CONFIG.WindowWidth,
    CONFIG.WindowHeight
)
Main.Position = UDim2.new(0.5, -325, 0.5, -215)
Main.BackgroundColor3 = CONFIG.Background
Main.BorderSizePixel = 0
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
HeaderTitle.Size = UDim2.new(1, -20, 1, 0)
HeaderTitle.Position = UDim2.fromOffset(14, 0)
HeaderTitle.BackgroundTransparency = 1
HeaderTitle.Text = "INSTANCE SCANNER  |  v0.3"
HeaderTitle.TextColor3 = CONFIG.Text
HeaderTitle.Font = Enum.Font.GothamBold
HeaderTitle.TextSize = 13
HeaderTitle.TextXAlignment = Enum.TextXAlignment.Left
HeaderTitle.Parent = Header

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
SearchBox.TextSize = 12
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
RefreshButton.TextSize = 10
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
Explorer.ScrollBarThickness = 6
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
PropertyTitle.TextSize = 12
PropertyTitle.TextXAlignment = Enum.TextXAlignment.Left
PropertyTitle.Parent = Properties

local PropertyInfo = Instance.new("TextLabel")
PropertyInfo.Size = UDim2.new(1, -16, 1, -84)
PropertyInfo.Position = UDim2.fromOffset(8, 36)
PropertyInfo.BackgroundTransparency = 1
PropertyInfo.Text = "Select an instance to inspect."
PropertyInfo.TextColor3 = CONFIG.Muted
PropertyInfo.Font = Enum.Font.Code
PropertyInfo.TextSize = 11
PropertyInfo.TextWrapped = true
PropertyInfo.TextYAlignment = Enum.TextYAlignment.Top
PropertyInfo.TextXAlignment = Enum.TextXAlignment.Left
PropertyInfo.Parent = Properties

local CopyButton = Instance.new("TextButton")
CopyButton.Size = UDim2.new(1, -16, 0, 28)
CopyButton.Position = UDim2.new(0, 8, 1, -36)
CopyButton.BackgroundColor3 = CONFIG.Accent
CopyButton.Text = "VIEW / COPY INSTANCE INFO"
CopyButton.TextColor3 = CONFIG.Text
CopyButton.Font = Enum.Font.GothamBold
CopyButton.TextSize = 10
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
CopyWindow.Size = UDim2.new(0, 430, 0, 270)
CopyWindow.Position = UDim2.new(0.5, -215, 0.5, -135)
CopyWindow.BackgroundColor3 = CONFIG.Background
CopyWindow.BorderSizePixel = 0
CopyWindow.Visible = false
CopyWindow.ZIndex = 5
CopyWindow.Parent = Main

local CopyWindowCorner = Instance.new("UICorner")
CopyWindowCorner.CornerRadius = UDim.new(0, 8)
CopyWindowCorner.Parent = CopyWindow

local CopyTitle = Instance.new("TextLabel")
CopyTitle.Size = UDim2.new(1, -50, 0, 35)
CopyTitle.Position = UDim2.fromOffset(12, 0)
CopyTitle.BackgroundTransparency = 1
CopyTitle.Text = "INSTANCE INFORMATION"
CopyTitle.TextColor3 = CONFIG.Text
CopyTitle.Font = Enum.Font.GothamBold
CopyTitle.TextSize = 12
CopyTitle.TextXAlignment = Enum.TextXAlignment.Left
CopyTitle.ZIndex = 6
CopyTitle.Parent = CopyWindow

local CloseCopy = Instance.new("TextButton")
CloseCopy.Size = UDim2.fromOffset(30, 27)
CloseCopy.Position = UDim2.new(1, -38, 0, 4)
CloseCopy.BackgroundColor3 = CONFIG.Secondary
CloseCopy.Text = "X"
CloseCopy.TextColor3 = CONFIG.Text
CloseCopy.Font = Enum.Font.GothamBold
CloseCopy.TextSize = 12
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
CopyTextBox.TextSize = 11
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
CopyInstructions.Size = UDim2.new(1, -24, 0, 30)
CopyInstructions.Position = UDim2.new(0, 12, 1, -36)
CopyInstructions.BackgroundTransparency = 1
CopyInstructions.Text = "Click the text, press Ctrl + A, then Ctrl + C."
CopyInstructions.TextColor3 = CONFIG.Muted
CopyInstructions.Font = Enum.Font.Gotham
CopyInstructions.TextSize = 10
CopyInstructions.TextXAlignment = Enum.TextXAlignment.Left
CopyInstructions.ZIndex = 6
CopyInstructions.Parent = CopyWindow

--==================================================
-- FOOTER
--==================================================

local Footer = Instance.new("TextLabel")
Footer.Size = UDim2.new(1, -28, 0, 20)
Footer.Position = UDim2.new(0, 14, 1, -25)
Footer.BackgroundTransparency = 1
Footer.Text = "Waiting for scan..."
Footer.TextColor3 = CONFIG.Muted
Footer.Font = Enum.Font.Gotham
Footer.TextSize = 10
Footer.TextXAlignment = Enum.TextXAlignment.Left
Footer.Parent = Main

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
            button.TextSize = 10
            button.TextXAlignment = Enum.TextXAlignment.Left
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

    task.defer(function()
        CopyTextBox:CaptureFocus()
        CopyTextBox.CursorPosition = 1
    end)

end)

CloseCopy.MouseButton1Click:Connect(function()

    CopyWindow.Visible = false
    CopyTextBox:ReleaseFocus()

end)

--==================================================
-- DRAGGABLE WINDOW
--==================================================

local dragging = false
local dragStart
local startPosition

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

            Main.Position = UDim2.new(
                startPosition.X.Scale,
                startPosition.X.Offset + delta.X,
                startPosition.Y.Scale,
                startPosition.Y.Offset + delta.Y
            )

        end

    end

end)

UserInputService.InputEnded:Connect(function(input)

    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = false

    end

end)

--==================================================
-- INITIAL SCAN
--==================================================

PerformScan()

print("[Instance Scanner v0.3] Loaded successfully.")
