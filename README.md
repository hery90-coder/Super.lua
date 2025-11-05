-- SUPER RING BOUNCE SCRIPT - GITHUB VERSION
-- Untuk digunakan dengan loadstring(game:HttpGet(...))
-- 
-- CARA PENGGUNAAN:
-- 1. Upload script ini ke GitHub sebagai raw file
-- 2. Di Roblox Studio, buat LocalScript atau Script di ring Part
-- 3. Gunakan: loadstring(game:HttpGet("URL_GITHUB_RAW_ANDA"))()

local SuperRingBounce = {}

-- ============================================
-- KONFIGURASI (Bisa disesuaikan)
-- ============================================
SuperRingBounce.Config = {
    BounceForce = 100,          -- Kekuatan pantulan
    BounceCooldown = 0.5,       -- Cooldown per objek (detik)
    GlowColor = Color3.fromRGB(0, 255, 255),  -- Warna ring
    ParticlesEnabled = true,    -- Aktifkan partikel
    SoundEnabled = true,        -- Aktifkan sound effect
    VisualEffects = true,       -- Efek visual saat bounce
    RingTransparency = 0.3,     -- Transparansi ring
    LightBrightness = 2,        -- Kecerahan glow
    LightRange = 20,            -- Jangkauan cahaya
}

-- ============================================
-- SISTEM BOUNCE
-- ============================================
local cooldowns = {}

function SuperRingBounce:GetBounceDirection(hitPart, ringCFrame)
    local ringLook = ringCFrame.LookVector
    local hitPos = hitPart.Position
    local ringPos = self.Ring.Position
    
    local direction = (hitPos - ringPos).Unit
    local dotProduct = direction:Dot(ringLook)
    
    if dotProduct > 0 then
        return ringLook
    else
        return -ringLook
    end
end

function SuperRingBounce:CreateVisualEffect(position)
    if not self.Config.VisualEffects then return end
    
    local effect = Instance.new("Part")
    effect.Shape = Enum.PartType.Ball
    effect.Material = Enum.Material.Neon
    effect.Color = self.Config.GlowColor
    effect.Size = Vector3.new(2, 2, 2)
    effect.Position = position
    effect.Anchored = true
    effect.CanCollide = false
    effect.Transparency = 0.5
    effect.Parent = workspace
    
    local tween = game:GetService("TweenService"):Create(
        effect,
        TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
        {Size = Vector3.new(6, 6, 6), Transparency = 1}
    )
    tween:Play()
    
    game:GetService("Debris"):AddItem(effect, 0.5)
end

function SuperRingBounce:PlayBounceSound()
    if not self.Config.SoundEnabled then return end
    
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://6315857815"
    sound.Volume = 0.5
    sound.Parent = self.Ring
    sound:Play()
    game:GetService("Debris"):AddItem(sound, 2)
end

function SuperRingBounce:FlashRing()
    local originalTransparency = self.Ring.Transparency
    self.Ring.Transparency = 0
    task.wait(0.1)
    self.Ring.Transparency = originalTransparency
end

function SuperRingBounce:BounceObject(hitPart)
    if not hitPart:IsA("BasePart") then return end
    if hitPart:IsGrounded() then return end
    if hitPart.Anchored then return end
    
    local now = tick()
    if cooldowns[hitPart] and now - cooldowns[hitPart] < self.Config.BounceCooldown then
        return
    end
    cooldowns[hitPart] = now
    
    local bounceDir = self:GetBounceDirection(hitPart, self.Ring.CFrame)
    
    local bodyVelocity = hitPart:FindFirstChildOfClass("BodyVelocity")
    if not bodyVelocity then
        bodyVelocity = Instance.new("BodyVelocity")
        bodyVelocity.MaxForce = Vector3.new(4e4, 4e4, 4e4)
        bodyVelocity.Parent = hitPart
    end
    
    bodyVelocity.Velocity = bounceDir * self.Config.BounceForce
    
    task.delay(0.3, function()
        if bodyVelocity and bodyVelocity.Parent then
            bodyVelocity:Destroy()
        end
    end)
    
    self:CreateVisualEffect(hitPart.Position)
    self:PlayBounceSound()
    self:FlashRing()
end

function SuperRingBounce:OnCharacterTouch(character)
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    if humanoidRootPart then
        self:BounceObject(humanoidRootPart)
    end
end

-- ============================================
-- SETUP VISUAL
-- ============================================
function SuperRingBounce:SetupVisuals()
    self.Ring.Material = Enum.Material.Neon
    self.Ring.Color = self.Config.GlowColor
    self.Ring.Transparency = self.Config.RingTransparency
    self.Ring.CanCollide = false
    
    -- PointLight
    local light = Instance.new("PointLight")
    light.Color = self.Config.GlowColor
    light.Brightness = self.Config.LightBrightness
    light.Range = self.Config.LightRange
    light.Parent = self.Ring
    
    -- ParticleEmitter
    if self.Config.ParticlesEnabled then
        local particles = Instance.new("ParticleEmitter")
        particles.Color = ColorSequence.new(self.Config.GlowColor)
        particles.LightEmission = 1
        particles.Size = NumberSequence.new(0.5)
        particles.Texture = "rbxasset://textures/particles/sparkles_main.dds"
        particles.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0),
            NumberSequenceKeypoint.new(1, 1)
        })
        particles.Lifetime = NumberRange.new(0.5, 1)
        particles.Rate = 20
        particles.Speed = NumberRange.new(2, 5)
        particles.SpreadAngle = Vector2.new(180, 180)
        particles.Parent = self.Ring
    end
end

-- ============================================
-- CLEANUP COOLDOWN
-- ============================================
function SuperRingBounce:StartCooldownCleanup()
    task.spawn(function()
        while self.Running do
            task.wait(5)
            local now = tick()
            for part, time in pairs(cooldowns) do
                if now - time > self.Config.BounceCooldown * 2 then
                    cooldowns[part] = nil
                end
            end
        end
    end)
end

-- ============================================
-- INISIALISASI
-- ============================================
function SuperRingBounce:Initialize(ringPart, customConfig)
    self.Ring = ringPart or script.Parent
    self.Running = true
    
    -- Merge custom config jika ada
    if customConfig then
        for key, value in pairs(customConfig) do
            self.Config[key] = value
        end
    end
    
    -- Setup
    self:SetupVisuals()
    self:StartCooldownCleanup()
    
    -- Event listener
    self.Ring.Touched:Connect(function(hit)
        local character = hit.Parent
        if character and character:FindFirstChild("Humanoid") then
            self:OnCharacterTouch(character)
        else
            self:BounceObject(hit)
        end
    end)
    
    print("✅ Super Ring Bounce initialized! Force:", self.Config.BounceForce)
    return self
end

function SuperRingBounce:Destroy()
    self.Running = false
    cooldowns = {}
    print("❌ Super Ring Bounce destroyed!")
end

-- ============================================
-- AUTO-EXECUTE (Jika script.Parent ada)
-- ============================================
if script.Parent and script.Parent:IsA("BasePart") then
    SuperRingBounce:Initialize(script.Parent)
end

-- Return module untuk penggunaan advanced
return SuperRingBounce
