local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")
local StarterGui = game:GetService("StarterGui")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- المتغيرات الأساسية للحماية
local protectionActive = false
local spamProtectionActive = false

local adminConnection = nil
local spamConnection = nil
local spamDescendantConnection = nil
local nvConnection = nil

local hiddenUIs = {}
local hiddenNightVision = {}

local targetKeywords = {"admin", "cmd", "log", "console", "command"}

-- الوصول لمجلد Assets الخاص بـ HDAdmin
local hdClient = ReplicatedStorage:WaitForChild("HDAdminHDClient", 10)
local assets = hdClient and hdClient:WaitForChild("Assets", 10) or nil

-- ==================== 1. وظائف NightVision (NV) ====================
local function hideNightVision(object)
    if object and object.Name == "NightVision" then
        if not table.find(hiddenNightVision, object) then
            table.insert(hiddenNightVision, object)
        end
        object.Parent = nil
    end
end

local function enableNVProtection()
    if not assets then return end

    nvConnection = assets.ChildAdded:Connect(function(object)
        if protectionActive then
            hideNightVision(object)
        end
    end)

    for _, object in ipairs(assets:GetChildren()) do
        hideNightVision(object)
    end
end

local function disableNVProtection()
    if nvConnection then
        nvConnection:Disconnect()
        nvConnection = nil
    end

    for _, object in ipairs(hiddenNightVision) do
        if object and object.Parent == nil and assets then
            object.Parent = assets
        end
    end
    hiddenNightVision = {}
end

-- ==================== 2. وظائف حماية الآدمن والـ Logs ====================
local function isTargetUI(gui)
    if not (gui:IsA("ScreenGui") or gui:IsA("Frame") or gui:IsA("BillboardGui")) then return false end
    local nameLower = gui.Name:lower()
    
    if nameLower == "guarduiparent" then return false end

    for _, keyword in ipairs(targetKeywords) do
        if nameLower:find(keyword) then
            return true
        end
    end
    return false
end

local function purgeAndHide(container)
    for _, child in ipairs(container:GetChildren()) do
        if isTargetUI(child) then
            if not hiddenUIs[child] then
                hiddenUIs[child] = child.Parent
                child.Parent = nil
            end
        end
    end
end

local function enableAdminProtection()
    protectionActive = true
    
    pcall(function() StarterGui:SetCore("DevConsoleVisible", false) end)
    purgeAndHide(PlayerGui)
    pcall(function() purgeAndHide(CoreGui) end)

    adminConnection = RunService.Heartbeat:Connect(function()
        if not protectionActive then return end
        pcall(function() StarterGui:SetCore("DevConsoleVisible", false) end)
        purgeAndHide(PlayerGui)
        pcall(function() purgeAndHide(CoreGui) end)
    end)

    enableNVProtection()
end

local function disableAdminProtection()
    protectionActive = false
    
    if adminConnection then
        adminConnection:Disconnect()
        adminConnection = nil
    end

    for gui, parent in pairs(hiddenUIs) do
        if gui and parent then
            gui.Parent = parent
        end
    end
    table.clear(hiddenUIs)

    disableNVProtection()
end

-- ==================== 3. وظائف إخفاء رسائل السبام (System) ====================
local function destroySystemNotifications()
    if not spamProtectionActive then return end

    for _, descendant in pairs(PlayerGui:GetDescendants()) do
        if descendant:IsA("TextLabel") and descendant.Text == "System" then
            local mainFrame = descendant.Parent
            if mainFrame and mainFrame:IsA("GuiObject") then
                pcall(function()
                    mainFrame.Visible = false
                    mainFrame:Destroy()
                end)
            end
        end
    end
end

local function enableSpamProtection()
    spamProtectionActive = true
    
    spamConnection = RunService.Heartbeat:Connect(function()
        if spamProtectionActive then
            destroySystemNotifications()
        end
    end)

    spamDescendantConnection = PlayerGui.DescendantAdded:Connect(function(descendant)
        if spamProtectionActive and descendant:IsA("TextLabel") and descendant.Text == "System" then
            local mainFrame = descendant.Parent
            if mainFrame and mainFrame:IsA("GuiObject") then
                pcall(function()
                    mainFrame.Visible = false
                    mainFrame:Destroy()
                end)
            end
        end
    end)
end

local function disableSpamProtection()
    spamProtectionActive = false
    
    if spamConnection then
        spamConnection:Disconnect()
        spamConnection = nil
    end
    if spamDescendantConnection then
        spamDescendantConnection:Disconnect()
        spamDescendantConnection = nil
    end
end

-- ==================== 4. إنشاء وديكور واجهة المستخدم (UI) ====================
local GuardUI = Instance.new("ScreenGui")
GuardUI.Name = "GuardUIParent"
GuardUI.ResetOnSpawn = false
GuardUI.Parent = PlayerGui

-- دالة تجعل أي Frame قابلة للسحب بسهولة (حركة اللائحة مثل الدائرة)
local function makeDraggable(guiObject)
    local dragging, dragInput, dragStart, startPos
    
    guiObject.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = guiObject.Position
            
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    
    guiObject.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            guiObject.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- الزر الميني المستدير (🛡️)
local MiniButton = Instance.new("TextButton")
MiniButton.Size = UDim2.new(0, 50, 0, 50)
MiniButton.Position = UDim2.new(0.02, 0, 0.4, 0)
MiniButton.BackgroundColor3 = Color3.fromRGB(15, 25, 18)
MiniButton.BackgroundTransparency = 0.15
MiniButton.Text = "🛡️"
MiniButton.TextSize = 24
MiniButton.Parent = GuardUI

local MiniCorner = Instance.new("UICorner")
MiniCorner.CornerRadius = UDim.new(1, 0)
MiniCorner.Parent = MiniButton

local MiniStroke = Instance.new("UIStroke")
MiniStroke.Color = Color3.fromRGB(46, 204, 113)
MiniStroke.Thickness = 2.5
MiniStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
MiniStroke.Parent = MiniButton

makeDraggable(MiniButton)

-- اللوحة الرئيسية
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 240, 0, 185)
MainFrame.Position = UDim2.new(0.02, 60, 0.4, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(12, 28, 16)
MainFrame.BackgroundTransparency = 0.25
MainFrame.Visible = true
MainFrame.Parent = GuardUI

local FrameCorner = Instance.new("UICorner")
FrameCorner.CornerRadius = UDim.new(0, 14)
FrameCorner.Parent = MainFrame

local FrameStroke = Instance.new("UIStroke")
FrameStroke.Color = Color3.fromRGB(46, 204, 113)
FrameStroke.Thickness = 2
FrameStroke.Parent = MainFrame

makeDraggable(MainFrame)

-- عنوان اللوحة
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 40)
Title.BackgroundTransparency = 1
Title.Text = "نظام الحماية 🛡️"
Title.TextColor3 = Color3.fromRGB(240, 255, 240)
Title.TextSize = 16
Title.Font = Enum.Font.GothamBold
Title.Parent = MainFrame

-- الزر الأول: حماية شاملة
local ToggleButton = Instance.new("TextButton")
ToggleButton.Size = UDim2.new(0.86, 0, 0, 38)
ToggleButton.Position = UDim2.new(0.07, 0, 0.26, 0)
ToggleButton.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
ToggleButton.Text = "حماية شاملة"
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.TextSize = 14
ToggleButton.Font = Enum.Font.GothamBold
ToggleButton.Parent = MainFrame

local BtnCorner1 = Instance.new("UICorner")
BtnCorner1.CornerRadius = UDim.new(0, 10)
BtnCorner1.Parent = ToggleButton

-- الزر الثاني: إخفاء رسائل سبام
local SpamButton = Instance.new("TextButton")
SpamButton.Size = UDim2.new(0.86, 0, 0, 38)
SpamButton.Position = UDim2.new(0.07, 0, 0.6, 0)
SpamButton.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
SpamButton.Text = "إخفاء رسائل سبام"
SpamButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SpamButton.TextSize = 14
SpamButton.Font = Enum.Font.GothamBold
SpamButton.Parent = MainFrame

local BtnCorner2 = Instance.new("UICorner")
BtnCorner2.CornerRadius = UDim.new(0, 10)
BtnCorner2.Parent = SpamButton

-- ==================== 5. الأحداث والتشغيل (Events) ====================
ToggleButton.MouseButton1Click:Connect(function()
    if not protectionActive then
        enableAdminProtection()
        ToggleButton.Text = "إيقاف الحماية الشاملة"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
    else
        disableAdminProtection()
        ToggleButton.Text = "حماية شاملة"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
    end
end)

SpamButton.MouseButton1Click:Connect(function()
    if not spamProtectionActive then
        enableSpamProtection()
        SpamButton.Text = "إيقاف إخفاء السبام"
        SpamButton.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
    else
        disableSpamProtection()
        SpamButton.Text = "إخفاء رسائل سبام"
        SpamButton.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
    end
end)

MiniButton.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
end)
