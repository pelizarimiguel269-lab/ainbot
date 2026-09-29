--[[
    ═══════════════════════════════════════════════════════
    PAINEL UNIFICADO - AIMBOT + ESP (Anti-Detecção)
    ═══════════════════════════════════════════════════════
    Técnicas aplicadas:
    - Ofuscação de strings (nomes de serviços/funções)
    - Randomização de nomes de variáveis/objetos
    - Delay aleatório no carregamento
    - Proteção contra hook de Drawing/loadstring
    - Anti-environment dump (bloqueia getfenv/setfenv de terceiros)
    - Heartbeat em vez de RenderStepped (menos suspeito)
    - Ocultação de metatable e hooks
]]

-- ═══════════════ CAMADA 1: OFUSCAÇÃO BÁSICA ═══════════════
local _0x = function(s)
    -- Decodifica strings ofuscadas (rot13 simples)
    return (s:gsub("%a", function(c)
        local b = c:byte()
        if b >= 65 and b <= 90 then return string.char((b - 65 + 13) % 26 + 65) end
        if b >= 97 and b <= 122 then return string.char((b - 97 + 13) % 26 + 97) end
        return c
    end))
end

-- Ofuscar acesso a serviços (dificulta detecção por string scan)
local _srv = game.GetService
local Players = _srv(game, _0x("Cynlref"))
local RunService = _srv(game, _0x("EhaFcnivpr"))
local UserInputService = _srv(game, _0x("HfreVachgFcnivpr"))
local Camera = workspace[_0x("PheeragPneren")] or workspace.CurrentCamera
local LocalPlayer = Players[_0x("YbpnyCynlre")]

-- ═══════════════ CAMADA 2: ANTI-DETECÇÃO ═══════════════

-- 2.1 - Bloquear leitura do ambiente por scripts externos
pcall(function()
    if getfenv and setfenv then
        local _env = getfenv(0)
        local _protect = newproxy(true)
        local _mt = getmetatable(_protect)
        mt.index = function(, k)
            if k == "getfenv" or k == "setfenv" or k == "debug" then
                return nil
            end
            return _env[k]
        end
    end
end)

-- 2.2 - Anti-hook de Drawing
local _Drawing = Drawing
pcall(function()
    local _realNew = Drawing.new
    Drawing.new = function(...)
        return _realNew(...)
    end
    -- Esconder referência global do hook
    if debug and debug.getinfo then
        local _info = debug.getinfo(1, "f")
        if _info and _info.func then
            pcall(function()
                debug.setinfo and debug.setinfo(_info.func, {source = "[C]", short_src = "[C]"})
            end)
        end
    end
end)

-- 2.3 - Delay aleatório (evita timing signature)
local _rand = math.random
task.wait(_rand(3, 12) / 10) -- 0.3s ~ 1.2s

-- 2.4 - Randomizar nomes de objetos criados
local function rndStr(len)
    local chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    local s = ""
    for _ = 1, (len or 8) do
        s = s .. chars:sub(_rand(1, #chars), _rand(1, #chars))
    end
    return s
end

-- 2.5 - Anti-dump: impede que outros scripts leiam nosso ambiente
pcall(function()
    local mt = getmetatable(_G)
    if mt then
        mt.__index = function(t, k)
            if k == "__SCRIPT_ENV" or k == "Config" or k == "ESPObjects" then
                return nil
            end
            return rawget(t, k)
        end
    end
end)

-- ═══════════════ CARREGAR RAYFIELD (com fallback) ═══════════════
local Rayfield
local _ok = pcall(function()
    Rayfield = loadstring(game:HttpGet(_0x("uggf://fvevhf.zrah/enlsvry")) )()
end)
if not _ok or not Rayfield then
    _ok = pcall(function()
        Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()
    end)
end
if not _ok then
    warn("[Painel] Falha ao carregar UI. Verifique sua internet/executor.")
    return
end

-- ═══════════════ VARIÁVEIS (nomes aleatórios) ═══════════════
local _cfg = {
    aim = false,          -- Aimbot ON/OFF
    fov = 100,            -- FOV do aimbot
    smooth = 5,           -- Suavidade
    teamAim = true,       -- Ignorar time no aimbot
    esp = false,          -- ESP ON/OFF
    name = true,          -- Nome no ESP
    box = true,           -- Caixa no ESP
    hp = false,           -- Vida no ESP
    dist = false,         -- Distância no ESP
    teamEsp = true,       -- Ignorar time no ESP
}

local _esp = {}           -- Armazena objetos ESP por player
local _target = nil       -- Alvo atual do aimbot

-- ═══════════════ FUNÇÕES AUXILIARES ═══════════════
local function alive(p)
    if not p or not p.Character then return false end
    local h = p.Character:FindFirstChildOfClass("Humanoid")
    return h and h.Health > 0
end

local function mate(p)
    if not p or not LocalPlayer then return false end
    return p.Team ~= nil and p.Team == LocalPlayer.Team
end

local function head(p)
    if not alive(p) then return nil end
    return p.Character:FindFirstChild("Head") or p.Character:FindFirstChild("HumanoidRootPart")
end

-- ═══════════════ AIMBOT FOV ═══════════════
local _fovCircle = Drawing.new("Circle")
_fovCircle.Thickness = 2
_fovCircle.Color = Color3.fromRGB(0, 170, 255)
_fovCircle.Transparency = 1
_fovCircle.Filled = false
_fovCircle.Visible = false
_fovCircle.NumSides = 64

local function updFov()
    if _cfg.aim then
        _fovCircle.Visible = true
        _fovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
        _fovCircle.Radius = _cfg.fov
    else
        _fovCircle.Visible = false
    end
end

local function getTarget()
    local close, short = nil, _cfg.fov
    local c = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and alive(p) and (not _cfg.teamAim or not mate(p)) then
            local h = head(p)
            if h then
                local sp, on = Camera:WorldToViewportPoint(h.Position)
                if on then
                    local d = (Vector2.new(sp.X, sp.Y) - c).Magnitude
                    if d < short then short, close = d, p end
                end
            end
        end
    end
    return close
end

local function aimAt(p)
    if not p or not alive(p) then return end
    local h = head(p)
    if not h then return end
    local camPos = Camera.CFrame.Position
    local newCF = CFrame.new(camPos, h.Position)
    if _cfg.smooth > 1 then
        Camera.CFrame = Camera.CFrame:Lerp(newCF, 1 / _cfg.smooth)
    else
        Camera.CFrame = newCF
    end
end

-- ═══════════════ ESP ═══════════════
local function createESP(p)
    if _esp[p] then return end
    local e = {
        b = Drawing.new("Square"),
        n = Drawing.new("Text"),
        h = Drawing.new("Text"),
        d = Drawing.new("Text"),
    }
    e.b.Thickness = 1
    e.b.Filled = false
    e.b.Transparency = 1
    e.b.Visible = false

    e.n.Size = 14; e.n.Center = true; e.n.Outline = true
    e.n.Color = Color3.fromRGB(255, 255, 255); e.n.Visible = false
    e.h.Size = 14; e.h.Center = true; e.h.Outline = true
    e.h.Color = Color3.fromRGB(0, 255, 0); e.h.Visible = false
    e.d.Size = 14; e.d.Center = true; e.d.Outline = true
    e.d.Color = Color3.fromRGB(255, 255, 0); e.d.Visible = false

    _esp[p] = e
end

local function removeESP(p)
    local e = _esp[p]
    if e then
        for _, o in pairs(e) do o:Remove() end
        _esp[p] = nil
    end
end

local function updESP()
    for p, e in pairs(_esp) do
        if _cfg.esp and alive(p) and p ~= LocalPlayer and (not _cfg.teamEsp or not mate(p)) then
            local h = head(p)
            local r = p.Character:FindFirstChild("HumanoidRootPart")
            local hm = p.Character:FindFirstChildOfClass("Humanoid")
            if h and r and hm then
                local hp, hon = Camera:WorldToViewportPoint(h.Position + Vector3.new(0, 1, 0))
                local rp, ron = Camera:WorldToViewportPoint(r.Position)
                if hon and ron then
                    local height = math.abs(hp.Y - rp.Y) * 2
                    local width = height * 0.6

                    if _cfg.box then
                        e.b.Size = Vector2.new(width, height)
                        e.b.Position = Vector2.new(hp.X - width / 2, hp.Y)
                        local pct = hm.Health / hm.MaxHealth
                        e.b.Color = Color3.fromRGB(255 * (1 - pct), 255 * pct, 0)
                        e.b.Visible = true
                    else e.b.Visible = false end

                    if _cfg.name then
                        e.n.Text = p.Name
                        e.n.Position = Vector2.new(hp.X, hp.Y - 20)
                        e.n.Visible = true
                    else e.n.Visible = false end

                    if _cfg.hp then
                        e.h.Text = string.format("%d HP", math.floor(hm.Health))
                        e.h.Position = Vector2.new(hp.X, hp.Y + height)
                        e.h.Visible = true
                    else e.h.Visible = false end

                    if _cfg.dist then
                        local dist = (Camera.CFrame.Position - r.Position).Magnitude
                        e.d.Text = string.format("%d m", math.floor(dist))
                        e.d.Position = Vector2.new(hp.X, hp.Y + height + 16)
                        e.d.Visible = true
                    else e.d.Visible = false end
                else
                    e.b.Visible = false; e.n.Visible = false
                    e.h.Visible = false; e.d.Visible = false
                end
            end
        else
            e.b.Visible = false; e.n.Visible = false
            e.h.Visible = false; e.d.Visible = false
        end
    end
end

-- ═══════════════ LOOP PRINCIPAL (Heartbeat = menos suspeito) ═══════════════
RunService.Heartbeat:Connect(function()
    updFov()
    updESP()
    if _cfg.aim then
        _target = getTarget()
        if _target then aimAt(_target) end
    end
end)

Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.wait(1)
        if _cfg.esp then createESP(p) end
    end)
end)
Players.PlayerRemoving:Connect(removeESP)

-- ═══════════════ UI - RAYFIELD (bordas arredondadas) ═══════════════
local _win = Rayfield:CreateWindow({
    Name = "⚡ Painel Aimbot + ESP ⚡",
    LoadingTitle = "Carregando...",
    LoadingSubtitle = "v1.0",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "PainelCfg",
        FileName = "Config"
    },
    KeySystem = false
})

-- Aba Aimbot
local tAim = _win:CreateTab("🎯 Aimbot", 4483362458)
tAim:CreateSection("Configurações")

tAim:CreateToggle({
    Name = "Ativar Aimbot",
    CurrentValue = false, Flag = "aim",
    Callback = function(v) _cfg.aim = v; if not v then _target = nil end end
})
tAim:CreateSlider({
    Name = "FOV do Aimbot", Range = {10, 500}, Increment = 5, Suffix = "px",
    CurrentValue = 100, Flag = "fov",
    Callback = function(v) _cfg.fov = v end
})
tAim:CreateSlider({
    Name = "Suavidade", Range = {1, 20}, Increment = 1, Suffix = "x",
    CurrentValue = 5, Flag = "smooth",
    Callback = function(v) _cfg.smooth = v end
})
tAim:CreateToggle({
    Name = "Ignorar Time", CurrentValue = true, Flag = "teamAim",
    Callback = function(v) _cfg.teamAim = v end
})

-- Aba ESP
local tEsp = _win:CreateTab("👁️ ESP", 4483362458)
tEsp:CreateSection("Configurações")

tEsp:CreateToggle({
    Name = "Ativar ESP", CurrentValue = false, Flag = "esp",
    Callback = function(v)
        _cfg.esp = v
        if v then
            for _, p in ipairs(Players:GetPlayers()) do
                if p ~= LocalPlayer then createESP(p) end
            end
        else
            for p in pairs(_esp) do removeESP(p) end
        end
    end
})
tEsp:CreateToggle({
    Name = "Mostrar Nome", CurrentValue = true, Flag = "name",
    Callback = function(v) _cfg.name = v end
})
tEsp:CreateToggle({
    Name = "Mostrar Caixa", CurrentValue = true, Flag = "box",
    Callback = function(v) _cfg.box = v end
})
tEsp:CreateToggle({
    Name = "Mostrar Vida", CurrentValue = false, Flag = "hp",
    Callback = function(v) _cfg.hp = v end
})
tEsp:CreateToggle({
    Name = "Mostrar Distância", CurrentValue = false, Flag = "dist",
    Callback = function(v) _cfg.dist = v end
})
tEsp:CreateToggle({
    Name = "Ignorar Time", CurrentValue = true, Flag = "teamEsp",
    Callback = function(v) _cfg.teamEsp = v end
})

-- Aba Config
local tCfg = _win:CreateTab("⚙️ Config", 4483362458)
tCfg:CreateButton({
    Name = "Destruir UI",
    Callback = function()
        Rayfield:Destroy()
        _fovCircle:Remove()
        for p in pairs(_esp) do removeESP(p) end
    end
})

Rayfield:LoadConfiguration()
