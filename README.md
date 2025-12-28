-- Executor Delta: Auto-equip skate quando fruto chega ao nível máximo + webhook + inversão/unequip + leaderstats
-- CONFIGURE AQUI:
local ASSET_ID = 123456789                 -- << ID do seu skate (modelo/mesh)
local SOUND_ASSET_ID = 184352171           -- << ID do som que toca ao equipar (opcional, number or nil)
local FRUTOS_PARENT_PATH = "workspace.Frutos" -- << caminho (ex: "workspace.Frutos" ou "ReplicatedStorage.Inventario")
local LEVEL_PROPERTY_NAME = "Nivel"        -- << nome do IntValue/NumberValue ou Attribute que guarda o level
local LEVEL_THRESHOLD = 3                  -- << nível de trigger (ou use seu nível máximo)
local AUTO_WELD_TO_HRP = true
local MARK_COLLECTED_NAME = "ColetadoPorExecutor"
local WEBHOOK_URL = "https://example.com/webhook" -- << coloque a URL do seu endpoint (ou "" para desativar)
local SCORE_INCREMENT = 1                  -- quanto incrementar na leaderstats ao coletar
local ATTACH_MODE = "back"                 -- "back" | "hip" | "feet" | "float"  (offset visual)
local DEBUG = true

-- NÃO PRECISA MEXER ABAIXO SE NÃO SOUBER O QUE FAZ
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local HttpService = game:GetService("HttpService")

local localPlayer = Players.LocalPlayer
if not localPlayer then
    warn("[ExecutorDelta] LocalPlayer não disponível. Execute no cliente.")
    return
end

local equippedModel = nil -- referência ao skate atualmente equipado

local function log(...)
    if DEBUG then print("[ExecutorDelta]", ...) end
end

local function resolvePath(path)
    if type(path) ~= "string" then return nil end
    local parts = {}
    for part in path:gmatch("[^%.]+") do table.insert(parts, part) end
    local rootName = table.remove(parts,1)
    local root
    if not rootName then return nil end
    local rlower = rootName:lower()
    if rlower == "workspace" then root = Workspace
    elseif rlower == "replicatedstorage" then root = ReplicatedStorage
    elseif rlower == "players" then root = Players
    else
        root = Workspace:FindFirstChild(rootName) or ReplicatedStorage:FindFirstChild(rootName) or Players:FindFirstChild(rootName)
    end
    if not root then return nil end
    local node = root
    for _,p in ipairs(parts) do
        node = node:FindFirstChild(p)
        if not node then return nil end
    end
    return node
end

local function safePcall(fn,...)
    local ok, res = pcall(fn, ...)
    return ok, res
end

local function sendWebhook(data)
    if not WEBHOOK_URL or WEBHOOK_URL == "" then return end
    local ok, err = pcall(function()
        local json = HttpService:JSONEncode(data)
        -- PostAsync pode falhar se HttpService estiver bloqueado
        HttpService:PostAsync(WEBHOOK_URL, json, Enum.HttpContentType.ApplicationJson)
    end)
    if not ok then
        warn("[ExecutorDelta] Falha ao enviar webhook:", err)
    else
        log("Webhook enviado")
    end
end

local function ensureLeaderstats()
    local ls = localPlayer:FindFirstChild("leaderstats")
    if not ls then
        ls = Instance.new("Folder")
        ls.Name = "leaderstats"
        ls.Parent = localPlayer
    end
    local score = ls:FindFirstChild("Score")
    if not score then
        score = Instance.new("IntValue")
        score.Name = "Score"
        score.Value = 0
        score.Parent = ls
    end
    return score
end

local function spawnSkateModel()
    log("Tentando carregar asset:", ASSET_ID)
    local model = nil
    local ok, res = pcall(function()
        return game:GetObjects("rbxassetid://" .. tostring(ASSET_ID))[1]
    end)
    if ok and res and typeof(res) == "Instance" then
        model = res
        model.Parent = Workspace
        model:MakeJoints()
        log("Modelo carregado via game:GetObjects")
        return model
    end

    -- fallback: criar Part com SpecialMesh
    local part = Instance.new("Part")
    part.Name = "ExecutorSkate"
    part.Size = Vector3.new(4, 0.25, 2)
    part.Anchored = false
    part.CanCollide = false
    part.Parent = Workspace

    local mesh = Instance.new("SpecialMesh", part)
    mesh.MeshType = Enum.MeshType.FileMesh
    mesh.MeshId = "rbxassetid://" .. tostring(ASSET_ID)
    log("Modelo fallback criado como Part + SpecialMesh")
    return part
end

local function getModelPrimary(model)
    if not model then return nil end
    if model:IsA("BasePart") then return model end
    if model.PrimaryPart then return model.PrimaryPart end
    for _,v in ipairs(model:GetDescendants()) do
        if v:IsA("BasePart") then return v end
    end
    return nil
end

local function equipSkateToPlayer(model, attachMode)
    if not model then return end
    local character = localPlayer.Character or localPlayer.CharacterAdded:Wait()
    local hrp = character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso") or character:FindFirstChildWhichIsA("BasePart")
    if not hrp then
        warn("[ExecutorDelta] HRP não encontrado para equipar skate.")
        return
    end

    local primary = getModelPrimary(model)
    if not primary then
        warn("[ExecutorDelta] Modelo não tem parte para weld.")
        return
    end

    -- Definir offsets conforme modo
    local offsetCFrame = CFrame.new(0, -2, 0) -- default
    attachMode = attachMode or ATTACH_MODE
    if attachMode == "back" then offsetCFrame = CFrame.new
# Nickwgx
