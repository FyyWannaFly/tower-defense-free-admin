# TOWER DEFENSE FREE ADMIN

Panduan cepat untuk membuat game Tower Defense yang bisa dimainkan di Roblox Studio dengan script siap pakai.

## 1. Buka Roblox Studio
- Buat project baru
- Pilih Baseplate
- Hapus Baseplate kalau mau

## 2. Bikin map basic
- Insert > Part
- Nama: Ground
- Size: 200, 1, 200
- Position: 0, 0, 0
- Anchored: true
- Material: Grass

Buat jalur enemy:
- Insert > Part
- Nama: Path
- Size: 20, 1, 8
- Color: dark gray
- Position: 0, 0.5, 0
- Anchored: true

Buat beberapa bagian path agar enemy bisa jalan.

## 3. Bikin spawn dan target
- Insert > SpawnLocation
- Letakkan di awal jalur
- Nama: SpawnPoint

- Insert > Part
- Nama: Goal
- Size: 6, 6, 6
- Position: ujung jalur
- Color: red
- Anchored: true

## 4. Buat turret tower
- Insert > Part
- Nama: Turret
- Size: 2, 5, 2
- Color: cyan
- Anchored: true

## 5. Bikin script utama di ServerScriptService
Klik kanan ServerScriptService > Script
Copy paste script berikut:

```lua
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local roundTime = 60
local startGold = 100
local enemySpeed = 6

local folder = Instance.new("Folder")
folder.Name = "TDGame"
folder.Parent = workspace

local enemyFolder = Instance.new("Folder")
enemyFolder.Name = "Enemies"
enemyFolder.Parent = folder

local towerFolder = Instance.new("Folder")
towerFolder.Name = "Towers"
towerFolder.Parent = folder

local roundValue = Instance.new("IntValue")
roundValue.Name = "Round"
roundValue.Value = 1
roundValue.Parent = folder

local goldValue = Instance.new("IntValue")
goldValue.Name = "Gold"
goldValue.Value = startGold
goldValue.Parent = folder

local livesValue = Instance.new("IntValue")
livesValue.Name = "Lives"
livesValue.Value = 20
livesValue.Parent = folder

local pathParts = workspace:GetDescendants()
local pathList = {}
for _, part in ipairs(pathParts) do
    if part.Name == "Path" then
        table.insert(pathList, part.Position)
    end
end

if #pathList == 0 then
    pathList = {
        Vector3.new(0, 1, -30),
        Vector3.new(0, 1, -10),
        Vector3.new(20, 1, -10),
        Vector3.new(20, 1, 20),
        Vector3.new(-20, 1, 20),
        Vector3.new(-20, 1, 0)
    }
end

local function createEnemy()
    local enemy = Instance.new("Part")
    enemy.Name = "Enemy"
    enemy.Shape = Enum.PartType.Cylinder
    enemy.Size = Vector3.new(2.5, 1.5, 2.5)
    enemy.Color = Color3.fromRGB(255, 80, 80)
    enemy.Material = Enum.Material.SmoothPlastic
    enemy.CanCollide = false
    enemy.Anchored = true
    enemy.Parent = enemyFolder

    local hp = Instance.new("IntValue")
    hp.Name = "HP"
    hp.Value = 30 + (roundValue.Value * 10)
    hp.Parent = enemy

    local positionIndex = 1
    enemy.Position = pathList[1]

    local mover = coroutine.create(function()
        while enemy.Parent do
            if positionIndex < #pathList then
                local target = pathList[positionIndex + 1]
                local direction = (target - enemy.Position)
                local distance = direction.Magnitude
                if distance < 1 then
                    positionIndex += 1
                else
                    enemy.Position = enemy.Position + (direction.Unit * enemySpeed)
                end
            else
                local goal = workspace:FindFirstChild("Goal")
                if goal then
                    local goalDiff = goal.Position - enemy.Position
                    if goalDiff.Magnitude < 5 then
                        livesValue.Value -= 1
                        enemy:Destroy()
                        break
                    end
                end
            end
            task.wait(0.05)
        end
    end)
    coroutine.resume(mover)

    return enemy
end

local function spawnWave()
    local enemyCount = 4 + roundValue.Value * 2
    for i = 1, enemyCount do
        task.delay(i * 0.8, createEnemy)
    end
end

local function findNearestEnemy(tower)
    local nearest = nil
    local nearestDist = math.huge
    for _, enemy in ipairs(enemyFolder:GetChildren()) do
        if enemy:IsA("Part") then
            local dist = (tower.Position - enemy.Position).Magnitude
            if dist < 28 and dist < nearestDist then
                nearestDist = dist
                nearest = enemy
            end
        end
    end
    return nearest
end

local function shootTower(tower)
    local target = findNearestEnemy(tower)
    if target then
        local direction = (target.Position - tower.Position)
        if direction.Magnitude <= 30 then
            local bullet = Instance.new("Part")
            bullet.Shape = Enum.PartType.Ball
            bullet.Size = Vector3.new(0.7, 0.7, 0.7)
            bullet.Color = Color3.fromRGB(255, 200, 0)
            bullet.Material = Enum.Material.Neon
            bullet.CanCollide = false
            bullet.Anchored = false
            bullet.Position = tower.Position + Vector3.new(0, 3, 0)
            bullet.Velocity = direction.Unit * 50
            bullet.Parent = workspace

            local connection
            connection = bullet.Touched:Connect(function(hit)
                if hit.Parent == target then
                    local hp = target:FindFirstChild("HP")
                    if hp then
                        hp.Value -= 10
                        if hp.Value <= 0 then
                            goldValue.Value += 10
                            target:Destroy()
                        end
                    end
                    bullet:Destroy()
                    connection:Disconnect()
                end
            end)
        end
    end
end

local function startTowerLoop()
    while true do
        for _, tower in ipairs(towerFolder:GetChildren()) do
            if tower:IsA("Part") then
                shootTower(tower)
            end
        end
        task.wait(0.6)
    end
end

local function createTower(position)
    local tower = Instance.new("Part")
    tower.Name = "Tower"
    tower.Size = Vector3.new(2, 5, 2)
    tower.Position = position
    tower.Color = Color3.fromRGB(100, 200, 255)
    tower.Material = Enum.Material.Metal
    tower.Anchored = true
    tower.Parent = towerFolder
end

local function onPlayerAdded(player)
    local leaderstats = player:FindFirstChild("leaderstats")
    if not leaderstats then
        leaderstats = Instance.new("Folder")
        leaderstats.Name = "leaderstats"
        leaderstats.Parent = player
    end

    local gold = leaderstats:FindFirstChild("Gold")
    if not gold then
        gold = Instance.new("IntValue")
        gold.Name = "Gold"
        gold.Value = startGold
        gold.Parent = leaderstats
    end

    local lives = leaderstats:FindFirstChild("Lives")
    if not lives then
        lives = Instance.new("IntValue")
        lives.Name = "Lives"
        lives.Value = 20
        lives.Parent = leaderstats
    end

    local round = leaderstats:FindFirstChild("Round")
    if not round then
        round = Instance.new("IntValue")
        round.Name = "Round"
        round.Value = 1
        round.Parent = leaderstats
    end
end

for _, player in ipairs(Players:GetPlayers()) do
    onPlayerAdded(player)
end
Players.PlayerAdded:Connect(onPlayerAdded)

local waveLoop = coroutine.create(function()
    while true do
        if livesValue.Value > 0 then
            spawnWave()
            roundValue.Value += 1
            for _, player in ipairs(Players:GetPlayers()) do
                local leaderstats = player:FindFirstChild("leaderstats")
                if leaderstats then
                    local roundStat = leaderstats:FindFirstChild("Round")
                    if roundStat then
                        roundStat.Value = roundValue.Value
                    end
                end
            end
        end
        task.wait(15)
    end
end)
coroutine.resume(waveLoop)

spawnWave()
startTowerLoop()
```

## 6. Buat script untuk build tower
Klik ServerScriptService > Script lagi
Nama: BuildTowerScript
Copy paste:

```lua
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")

local towerCost = 25
local towerFolder = workspace:FindFirstChild("TDGame") and workspace.TDGame:FindFirstChild("Towers")

local function onClick(player)
    local character = player.Character
    if not character then return end

    local leaderstats = player:FindFirstChild("leaderstats")
    if not leaderstats then return end

    local gold = leaderstats:FindFirstChild("Gold")
    if not gold then return end

    if gold.Value < towerCost then
        print("Not enough gold")
        return
    end

    local mouse = player:GetMouse()
    local targetPos = mouse.Hit.Position
    local tower = Instance.new("Part")
    tower.Name = "Tower"
    tower.Size = Vector3.new(2, 5, 2)
    tower.Color = Color3.fromRGB(100, 200, 255)
    tower.Material = Enum.Material.Metal
    tower.Position = targetPos + Vector3.new(0, 3, 0)
    tower.Anchored = true
    tower.Parent = workspace.TDGame.Towers

    gold.Value -= towerCost
end

for _, player in ipairs(Players:GetPlayers()) do
    player:GetMouse().Button1Down:Connect(function()
        onClick(player)
    end)
end

Players.PlayerAdded:Connect(function(player)
    player:GetMouse().Button1Down:Connect(function()
        onClick(player)
    end)
end)
```

## 7. Bikin UI score dan gold
Di StarterGui > ScreenGui
Buat 3 TextLabel:
- Gold
- Lives
- Round

Script LocalScript:

```lua
local player = game.Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "TDUI"
screenGui.Parent = playerGui

local goldLabel = Instance.new("TextLabel")
goldLabel.Size = UDim2.new(0, 220, 0, 40)
goldLabel.Position = UDim2.new(0, 20, 0, 20)
goldLabel.Text = "Gold: 0"
goldLabel.BackgroundTransparency = 1
goldLabel.TextColor3 = Color3.fromRGB(255, 210, 50)
goldLabel.TextSize = 24
goldLabel.Font = Enum.Font.GothamBold
goldLabel.Parent = screenGui

local livesLabel = Instance.new("TextLabel")
livesLabel.Size = UDim2.new(0, 220, 0, 40)
livesLabel.Position = UDim2.new(0, 20, 0, 60)
livesLabel.Text = "Lives: 0"
livesLabel.BackgroundTransparency = 1
goldLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
livesLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
livesLabel.TextSize = 24
goldLabel.Font = Enum.Font.GothamBold
livesLabel.Font = Enum.Font.GothamBold
livesLabel.Parent = screenGui

local roundLabel = Instance.new("TextLabel")
roundLabel.Size = UDim2.new(0, 220, 0, 40)
roundLabel.Position = UDim2.new(0, 20, 0, 100)
roundLabel.Text = "Round: 1"
roundLabel.BackgroundTransparency = 1
roundLabel.TextColor3 = Color3.fromRGB(100, 255, 150)
roundLabel.TextSize = 24
roundLabel.Font = Enum.Font.GothamBold
roundLabel.Parent = screenGui

while true do
    local leaderstats = player:FindFirstChild("leaderstats")
    if leaderstats then
        local gold = leaderstats:FindFirstChild("Gold")
        local lives = leaderstats:FindFirstChild("Lives")
        local round = leaderstats:FindFirstChild("Round")

        if gold then
            goldLabel.Text = "Gold: " .. gold.Value
        end
        if lives then
            livesLabel.Text = "Lives: " .. lives.Value
        end
        if round then
            roundLabel.Text = "Round: " .. round.Value
        end
    end
    task.wait(0.1)
end
```

## 8. Mainkan game
- Tekan Play di Roblox Studio
- Klik tombol kiri mouse di area map untuk menempatkan tower
- Tower otomatis menembak musuh
- Kalahkan wave enemy
- Kumpulkan gold
- Survive sampai mungkin game over

## 9. Bonus
Kalau mau, nanti saya bisa bikin versi yang lebih advanced:
- Tower upgrade
- Menu buy tower
- Enemy lebih banyak
- Boss wave
- Shop system
- Admin commands

Tapi ini versi paling simpel yang bisa langsung dipakai.
