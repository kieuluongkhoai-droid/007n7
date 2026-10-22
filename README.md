if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Biến quản lý trạng thái
local CloneModel = nil
local IsControlClone = false -- false: Real Body hoạt động | true: Clone hoạt động
local CloneHumanoid = nil
local CloneHRP = nil

-- Dọn dẹp UI cũ nếu có
pcall(function() 
    if CoreGui:FindFirstChild("CloneSystemUI") then 
        CoreGui.CloneSystemUI:Destroy() 
    end 
end)

----------------------------------------------------------------
-- 1. GIAO DIỆN MENU (UI) CÓ NÚT THU NHỎ
----------------------------------------------------------------
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CloneSystemUI"
ScreenGui.ResetOnSpawn = false
pcall(function() ScreenGui.Parent = CoreGui end)
if not ScreenGui.Parent then ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui") end

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 240, 0, 200)
MainFrame.Position = UDim2.new(0.05, 0, 0.2, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui

local UIStroke = Instance.new("UIStroke")
UIStroke.Thickness = 2
UIStroke.Color = Color3.fromRGB(0, 255, 180)
UIStroke.Parent = MainFrame
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 10)

-- Thanh Tiêu Đề
local TitleLabel = Instance.new("TextLabel", MainFrame)
TitleLabel.Size = UDim2.new(1, -35, 0, 35)
TitleLabel.Text = "  CLONE SWAPPER SYSTEM"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.TextSize = 11
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.BackgroundColor3 = Color3.fromRGB(28, 28, 38)
Instance.new("UICorner", TitleLabel).CornerRadius = UDim.new(0, 10)

-- Nút Thu Nhỏ [-]
local MinimizeBtn = Instance.new("TextButton", MainFrame)
MinimizeBtn.Size = UDim2.new(0, 35, 0, 35)
MinimizeBtn.Position = UDim2.new(1, -35, 0, 0)
MinimizeBtn.Text = "-"
MinimizeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MinimizeBtn.TextSize = 18
MinimizeBtn.Font = Enum.Font.GothamBold
MinimizeBtn.BackgroundColor3 = Color3.fromRGB(38, 38, 50)
Instance.new("UICorner", MinimizeBtn).CornerRadius = UDim.new(0, 10)

local function createBtn(text, y, color)
    local btn = Instance.new("TextButton", MainFrame)
    btn.Size = UDim2.new(0, 210, 0, 30)
    btn.Position = UDim2.new(0, 15, 0, y)
    btn.Text = text
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 10
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.BackgroundColor3 = color or Color3.fromRGB(35, 35, 48)
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    return btn
end

local BtnSpawnClone = createBtn("👤 Tạo Clone", 42, Color3.fromRGB(0, 150, 120))
local BtnSwapCtrl   = createBtn("🔄 SWAP Góc Nhìn (Real Body)", 77, Color3.fromRGB(40, 40, 55))
local BtnTeleport   = createBtn("🔀 Đổi Chỗ Vị Trí (TP Swap)", 112, Color3.fromRGB(180, 100, 0))
local BtnRemove     = createBtn("❌ Xóa Clone", 147, Color3.fromRGB(150, 40, 40))

----------------------------------------------------------------
-- 2. HÀM TẠO VÀ XÓA CLONE (GIỮ NGUYÊN ANIMATION SCRIPT)
----------------------------------------------------------------
local function RemoveClone()
    if CloneModel then
        CloneModel:Destroy()
        CloneModel = nil
        CloneHumanoid = nil
        CloneHRP = nil
    end
    IsControlClone = false
    BtnSwapCtrl.Text = "🔄 SWAP Góc Nhìn (Real Body)"
    BtnSwapCtrl.BackgroundColor3 = Color3.fromRGB(40, 40, 55)

    -- Trả lại quyền điều khiển hoàn toàn cho Real Body
    local realChar = LocalPlayer.Character
    if realChar and realChar:FindFirstChildOfClass("Humanoid") then
        local realHum = realChar:FindFirstChildOfClass("Humanoid")
        realHum.WalkSpeed = 16
        realHum.JumpPower = 50
        Camera.CameraSubject = realHum
    end
end

local function SpawnClone()
    local realChar = LocalPlayer.Character
    if not realChar or not realChar:FindFirstChild("HumanoidRootPart") then return end

    RemoveClone()

    realChar.Archivable = true
    CloneModel = realChar:Clone()
    realChar.Archivable = false

    CloneModel.Name = LocalPlayer.Name .. "_Clone"
    CloneModel.Parent = workspace

    CloneHumanoid = CloneModel:FindFirstChildOfClass("Humanoid")
    CloneHRP = CloneModel:FindFirstChild("HumanoidRootPart")

    -- Sao chép Animate script từ Real Body sang Clone để giữ nguyên Animation
    local animateScript = realChar:FindFirstChild("Animate")
    if animateScript then
        local cloneAnimate = animateScript:Clone()
        cloneAnimate.Parent = CloneModel
    end

    -- Đặt vị trí Clone kế bên Real Body
    if CloneHRP and realChar:FindFirstChild("HumanoidRootPart") then
        CloneHRP.CFrame = realChar.HumanoidRootPart.CFrame * CFrame.new(3, 0, 0)
    end

    -- Khóa cứng Clone lúc mới tạo ra
    if CloneHumanoid then
        CloneHumanoid.WalkSpeed = 0
        CloneHumanoid.JumpPower = 0
    end
end

----------------------------------------------------------------
-- 3. XỬ LÝ ĐIỀU KHIỂN & TRUYỀN DI CHUYỂN / ANIMATION
----------------------------------------------------------------
RunService.RenderStepped:Connect(function()
    local realChar = LocalPlayer.Character
    if not realChar or not realChar:FindFirstChildOfClass("Humanoid") then return end
    local realHum = realChar:FindFirstChildOfClass("Humanoid")

    if IsControlClone and CloneModel and CloneHumanoid then
        -- 1. Khóa cứng Real Body tại vị trí hiện tại
        realHum.WalkSpeed = 0
        realHum.JumpPower = 0

        -- 2. Mở khóa Clone & Đọc đầu vào di chuyển từ Joystick/Phím
        CloneHumanoid.WalkSpeed = 16
        CloneHumanoid.JumpPower = 50

        -- Truyền hướng đi và lệnh nhảy từ cần điều khiển sang Clone
        CloneHumanoid:Move(realHum.MoveDirection, false)
        if realHum.Jump then
            CloneHumanoid.Jump = true
        end

        -- 3. Chuyển Camera sang Clone
        Camera.CameraSubject = CloneHumanoid

    else
        -- 1. Mở khóa di chuyển cho Real Body
        realHum.WalkSpeed = 16
        realHum.JumpPower = 50

        -- 2. Khóa cứng Clone đứng yên tại vị trí hiện tại
        if CloneHumanoid then
            CloneHumanoid.WalkSpeed = 0
            CloneHumanoid.JumpPower = 0
            CloneHumanoid:Move(Vector3.zero, false)
        end

        -- 3. Chuyển Camera về Real Body
        Camera.CameraSubject = realHum
    end
end)

----------------------------------------------------------------
-- 4. SỰ KIỆN NÚT BẤM (BUTTON EVENTS)
----------------------------------------------------------------
local isMinimized = false
MinimizeBtn.MouseButton1Click:Connect(function()
    isMinimized = not isMinimized
    if isMinimized then
        MainFrame.Size = UDim2.new(0, 240, 0, 35)
        MinimizeBtn.Text = "+"
    else
        MainFrame.Size = UDim2.new(0, 240, 0, 200)
        MinimizeBtn.Text = "-"
    end
end)

BtnSpawnClone.MouseButton1Click:Connect(function()
    SpawnClone()
end)

-- Nút Swap Góc Nhìn
BtnSwapCtrl.MouseButton1Click:Connect(function()
    if not CloneModel then return end

    IsControlClone = not IsControlClone

    if IsControlClone then
        BtnSwapCtrl.Text = "🔄 SWAP Góc Nhìn (Clone)"
        BtnSwapCtrl.BackgroundColor3 = Color3.fromRGB(0, 150, 200)
    else
        BtnSwapCtrl.Text = "🔄 SWAP Góc Nhìn (Real Body)"
        BtnSwapCtrl.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
    end
end)

-- Nút Đổi Chỗ Vị Trí (Teleport Swap vị trí Real Body & Clone)
BtnTeleport.MouseButton1Click:Connect(function()
    local realChar = LocalPlayer.Character
    if not realChar or not realChar:FindFirstChild("HumanoidRootPart") or not CloneHRP then return end

    local realHRP = realChar.HumanoidRootPart
    
    -- Tráo đổi tọa độ CFrame giữa Real Body và Clone
    local realPos = realHRP.CFrame
    local clonePos = CloneHRP.CFrame

    realHRP.CFrame = clonePos
    CloneHRP.CFrame = realPos
end)

BtnRemove.MouseButton1Click:Connect(function()
    RemoveClone()
end)

-- Phím tắt [INSERT] để ẩn/hiện UI
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        MainFrame.Visible = not MainFrame.Visible
    end
end)

print("Clone Swapper V11 Loaded Successfully!")
