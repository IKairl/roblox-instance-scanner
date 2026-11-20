
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
