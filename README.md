-- DevMoneyServer.lua (ServerScriptService)
-- Bezpieczny system dawania pieniędzy tylko dla zaufanych developerów (whitelist)

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local event = ReplicatedStorage:WaitForChild("DevGrantMoneyEvent")

-- KONFIGURACJA (wstaw tu swoje UserId deweloperów)
local DEV_WHITELIST = {
    [12345678] = true, -- przykładowy UserId: zamień na prawdziwe
    [87654321] = true,
}

local MAX_PER_REQUEST = 10000      -- maksymalna kwota na pojedyncze żądanie
local MAX_PER_MINUTE = 50000       -- maksymalnie na minutę na jednego dewelopera

-- Prosty rate limiter (per-user)
local recentAmount = {} -- [userId] = {amountThisMinute = X, timestampStart = os.time()}

local function canGrant(userId, amount)
    if not DEV_WHITELIST[userId] then
        return false, "Nie jesteś na whitelist."
    end
    if amount <= 0 then
        return false, "Kwota musi być większa od 0."
    end
    if amount > MAX_PER_REQUEST then
        return false, "Kwota przekracza maksymalną dozwoloną na jedno żądanie."
    end

    local now = os.time()
    local data = recentAmount[userId]
    if not data or now - data.timestampStart >= 60 then
        -- reset okresu
        recentAmount[userId] = { amountThisMinute = 0, timestampStart = now }
        data = recentAmount[userId]
    end

    if data.amountThisMinute + amount > MAX_PER_MINUTE then
        return false, "Przekroczenie limitu na minutę."
    end

    return true, ""
end

-- Przykładowa funkcja przyznawania waluty (dostosuj do twojego systemu ekonomii)
local function grantMoneyToPlayer(targetPlayer, amount)
    -- Tutaj musisz dodać swoją logikę zapisu waluty (leaderstats, DataStore, itp.)
    -- Poniżej przykład prosty: leaderstats.Cash
    if not targetPlayer then return false, "Gracz nie istnieje." end

    local leaderstats = targetPlayer:FindFirstChild("leaderstats")
    if not leaderstats then
        -- opcjonalnie tworzymy leaderstats jeśli go nie ma
        leaderstats = Instance.new("Folder")
        leaderstats.Name = "leaderstats"
        leaderstats.Parent = targetPlayer
    end

    local cash = leaderstats:FindFirstChild("Cash")
    if not cash then
        cash = Instance.new("IntValue")
        cash.Name = "Cash"
        cash.Value = 0
        cash.Parent = leaderstats
    end

    cash.Value = cash.Value + amount
    return true, "Przyznano " .. tostring(amount) .. " do " .. targetPlayer.Name
end

-- Obsługa remote
event.OnServerEvent:Connect(function(player, targetUserId, amountRequested)
    -- player = wysyłający żądanie deweloper
    -- targetUserId = UserId gracza, który ma otrzymać pieniądze
    -- amountRequested = liczba

    local senderId = player.UserId
    local amount = math.floor(tonumber(amountRequested) or 0)
    local targetId = tonumber(targetUserId) or player.UserId -- domyślnie do siebie

    local ok, reason = canGrant(senderId, amount)
    if not ok then
        warn("DevGrantMoneyEvent: próba od " .. player.Name .. " - odrzucono: " .. reason)
        return
    end

    -- znajdź docelowego gracza
    local targetPlayer = Players:GetPlayerByUserId(targetId)
    if not targetPlayer then
        warn("DevGrantMoneyEvent: nie znaleziono gracza o UserId " .. tostring(targetId))
        return
    end

    -- zaktualizuj limiter
    local now = os.time()
    local data = recentAmount[senderId]
    if not data or now - data.timestampStart >= 60 then
        recentAmount[senderId] = { amountThisMinute = amount, timestampStart = now }
    else
        data.amountThisMinute = data.amountThisMinute + amount
    end

    -- wykonaj przyznanie (tu: przykładowy)
    local success, msg = grantMoneyToPlayer(targetPlayer, amount)
    if success then
        print("DevGrantMoneyEvent: " .. player.Name .. " przyznał " .. tostring(amount) .. " do " .. targetPlayer.Name)
    else
        warn("DevGrantMoneyEvent błąd: " .. tostring(msg))
    end
end)
