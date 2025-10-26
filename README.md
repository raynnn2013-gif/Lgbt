-- LocalScript (colocar em StarterPlayerScripts)
-- Funções: Toggle Fly (F), Sprint (LeftShift hold), Jump Boost (toggle with J)
-- Uso: apenas em seu próprio jogo/ambiente de desenvolvimento.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer

local flying = false
local flySpeed = 50 -- velocidade de voo
local normalWalkSpeed = 16
local sprintMultiplier = 2.2
local sprinting = false
local jumpBoostEnabled = false
local jumpPowerDefault = 50
local jumpPowerBoost = 90

local function setupCharacter(char)
	local humanoid = char:WaitForChild("Humanoid")
	local hrp = char:WaitForChild("HumanoidRootPart")

	-- garantir configurações iniciais
	humanoid.WalkSpeed = normalWalkSpeed
	humanoid.UseJumpPower = true
	humanoid.JumpPower = jumpPowerDefault

	-- remover possíveis forces antigos
	for _, v in ipairs(hrp:GetChildren()) do
		if v.Name == "FlyBodyVelocity" or v.Name == "FlyBodyGyro" then
			v:Destroy()
		end
	end
end

local function enableFly(char)
	if flying then return end
	flying = true
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end

	local bv = Instance.new("BodyVelocity")
	bv.Name = "FlyBodyVelocity"
	bv.MaxForce = Vector3.new(1e5, 1e5, 1e5)
	bv.Velocity = Vector3.new(0,0,0)
	bv.Parent = hrp

	local bg = Instance.new("BodyGyro")
	bg.Name = "FlyBodyGyro"
	bg.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
	bg.CFrame = hrp.CFrame
	bg.Parent = hrp

	-- atualiza velocidade por frame
	local conn
	conn = RunService.RenderStepped:Connect(function()
		if not flying or not hrp.Parent then
			conn:Disconnect()
			if bv then bv:Destroy() end
			if bg then bg:Destroy() end
			return
		end

		-- direção baseada na câmera
		local cam = workspace.CurrentCamera
		local camCFrame = cam.CFrame
		local moveVector = Vector3.new(0,0,0)

		-- WSAD input
		local forward = UserInputService:IsKeyDown(Enum.KeyCode.W) and 1 or 0
		local backward = UserInputService:IsKeyDown(Enum.KeyCode.S) and 1 or 0
		local left = UserInputService:IsKeyDown(Enum.KeyCode.A) and 1 or 0
		local right = UserInputService:IsKeyDown(Enum.KeyCode.D) and 1 or 0
		local up = UserInputService:IsKeyDown(Enum.KeyCode.E) and 1 or 0
		local down = UserInputService:IsKeyDown(Enum.KeyCode.Q) and 1 or 0

		local z = forward - backward
		local x = right - left
		local y = up - down

		local forwardVec = Vector3.new(camCFrame.lookVector.X, 0, camCFrame.lookVector.Z).Unit
		if forwardVec ~= forwardVec then forwardVec = Vector3.new(0,0,-1) end -- fallback
		local rightVec = camCFrame.rightVector
		rightVec = Vector3.new(rightVec.X, 0, rightVec.Z).Unit
		if rightVec ~= rightVec then rightVec = Vector3.new(1,0,0) end

		moveVector = (forwardVec * z + rightVec * x) + Vector3.new(0, y, 0)

		if moveVector.Magnitude > 0 then
			bv.Velocity = moveVector.Unit * flySpeed
			bg.CFrame = CFrame.new(hrp.Position, hrp.Position + camCFrame.LookVector)
		else
			bv.Velocity = Vector3.new(0,0,0)
		end
	end)
end

local function disableFly(char)
	flying = false
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if hrp then
		local bv = hrp:FindFirstChild("FlyBodyVelocity")
		local bg = hrp:FindFirstChild("FlyBodyGyro")
		if bv then bv:Destroy() end
		if bg then bg:Destroy() end
	end
end

-- responde a trocas de personagem
player.CharacterAdded:Connect(function(char)
	setupCharacter(char)
end)

-- se já existir personagem
if player.Character then
	setupCharacter(player.Character)
end

-- input: F para alternar fly, LeftShift para sprint (hold), J para alternar jump boost
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	if input.KeyCode == Enum.KeyCode.F then
		local char = player.Character
		if not char then return end
		if flying then
			disableFly(char)
		else
			enableFly(char)
		end
	elseif input.KeyCode == Enum.KeyCode.LeftShift then
		sprinting = true
		local h = player.Character and player.Character:FindFirstChild("Humanoid")
		if h then h.WalkSpeed = normalWalkSpeed * sprintMultiplier end
	elseif input.KeyCode == Enum.KeyCode.J then
		-- toggle jump boost
		jumpBoostEnabled = not jumpBoostEnabled
		local h = player.Character and player.Character:FindFirstChild("Humanoid")
		if h then
			if jumpBoostEnabled then
				h.JumpPower = jumpPowerBoost
			else
				h.JumpPower = jumpPowerDefault
			end
		end
	end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	if input.KeyCode == Enum.KeyCode.LeftShift then
		sprinting = false
		local h = player.Character and player.Character:FindFirstChild("Humanoid")
		if h then h.WalkSpeed = normalWalkSpeed end
	end
end)

-- limpar forças se personagem morrer
Players.LocalPlayer.CharacterRemoving:Connect(function(char)
	disableFly(char)
end)
