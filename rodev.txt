

local ServerStorage = game:GetService("ServerStorage")
local TweenService = game:GetService("TweenService")

local spawnPortal = workspace:WaitForChild("Portal1")
local despawnPortal = workspace:WaitForChild("Portal2")


local SPAWN_INTERVAL = 2
local SPEED = 10
local FLOAT_HEIGHT = 4


local blockNames = {
	"PortalBlock",
	"PortalBlock2",
	"PortalBlock3",
	"PortalBlock4"
}

local function createTween(part, target)
	local distance = (target.Position - part.Position).Magnitude
	local time = distance / SPEED
	local info = TweenInfo.new(time, Enum.EasingStyle.Linear)
	return TweenService:Create(part, info, { Position = target.Position })
end

local function getBaseTargets()
	local targets = {}
	for _, obj in ipairs(workspace:GetDescendants()) do
		if obj:IsA("BasePart") and obj.Name == "TargetRowForBlocks" then
			table.insert(targets, obj)
		end
	end
	return targets
end


for _, target in ipairs(getBaseTargets()) do
	target:SetAttribute("Occupied", false)
end


local function sendBlockToBase(block)
	local targets = getBaseTargets()

	-- Find a FREE target
	local freeTarget = nil
	for _, target in ipairs(targets) do
		if not target:GetAttribute("Occupied") then
			freeTarget = target
			break
		end
	end

	if not freeTarget then
		warn("ALL TargetRowForBlocks ARE FULL")
		return
	end

	-- Mark target as occupied
	freeTarget:SetAttribute("Occupied", true)

	-- Lock block in place (NO FALLING)
	block.Anchored = true
	block.CanCollide = false
	block.CFrame = freeTarget.CFrame + Vector3.new(0, FLOAT_HEIGHT, 0)
end


local function spawnBlock()
	local name = blockNames[math.random(#blockNames)]
	local template = ServerStorage:FindFirstChild(name)
	if not template then return end

	local price = template:GetAttribute("Price")
	if not price then return end

	local block = template:Clone()
	block.Parent = workspace
	block.CFrame = spawnPortal.CFrame
	block.Anchored = true
	block.CanCollide = false

	-- Billboard GUI
	local gui = Instance.new("BillboardGui", block)
	gui.Size = UDim2.new(0, 130, 0, 40)
	gui.StudsOffset = Vector3.new(0, 3, 0)
	gui.AlwaysOnTop = true

	local label = Instance.new("TextLabel", gui)
	label.Size = UDim2.new(1, 0, 1, 0)
	label.BackgroundTransparency = 1
	label.TextScaled = true
	label.Font = Enum.Font.LuckiestGuy
	label.TextStrokeTransparency = 0
	label.TextColor3 = Color3.fromRGB(255, 221, 0)
	label.Text = price .. " Coins"


	local prompt = Instance.new("ProximityPrompt", block)
	prompt.ActionText = "Buy"
	prompt.ObjectText = "Block"
	prompt.HoldDuration = 0.75
	prompt.MaxActivationDistance = 20
	prompt.RequiresLineOfSight = false

	local bought = false
	local moveTween = createTween(block, despawnPortal)
	moveTween:Play()

	prompt.Triggered:Connect(function(player)
		if bought then return end

		local stats = player:FindFirstChild("leaderstats")
		local coins = stats and stats:FindFirstChild("Coins")
		if not coins or coins.Value < price then return end

		coins.Value -= price
		bought = true

		prompt:Destroy()
		label.Text = "BOUGHT"
		moveTween:Cancel()

		sendBlockToBase(block)
	end)

	moveTween.Completed:Connect(function()
		if not bought then
			block:Destroy()
		end
	end)
end


while true do
	spawnBlock()
	task.wait(SPAWN_INTERVAL)
end
