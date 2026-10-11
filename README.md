-- ARMA DE PORTAL v10
-- Clique/toque onde quer o portal. Portais em pe, ovais, estilo redemoinho verde.
-- 1o clique = entrada, 2o = saida, 3o substitui o primeiro, e assim por diante.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local lp = Players.LocalPlayer
local mouse = lp:GetMouse()

local ALTURA = 7      -- portal em pe
local LARGURA = 6.6   -- mais redondo, igual a imagem dourada
local COOLDOWN = 0.6
local ALCANCE = 600
local DIST_AR = 40     -- distancia do portal quando voce clica no ar/ceu (studs)
local PPS = 50        -- qualidade do desenho do portal

local COR_PORTAL = Color3.fromRGB(255, 195, 40) -- cor dourada do portal (mude aqui)
local VERDE_MEIO = COR_PORTAL
local BANDAS = { Color3.fromRGB(15, 115, 30), Color3.fromRGB(28, 150, 38), Color3.fromRGB(20, 130, 34) }
local LINHAS = { Color3.fromRGB(150, 245, 55), Color3.fromRGB(195, 255, 85), Color3.fromRGB(120, 225, 45) }
local BORDAS = { Color3.fromRGB(95, 225, 30), Color3.fromRGB(135, 240, 40), Color3.fromRGB(70, 205, 35) }


-- ========== IMAGEM DO PORTAL ==========
local ID_IMAGEM = 127017929001925   -- imagem do portal dourado
local IMAGEM_FINAL = "rbxassetid://" .. ID_IMAGEM

-- Se o ID for de um Decal, pega o ID real da imagem dentro dele
pcall(function()
    local obj = game:GetObjects("rbxassetid://" .. ID_IMAGEM)[1]
    if not obj then return end
    local achado
    if obj:IsA("Decal") or obj:IsA("Texture") then achado = obj.Texture
    elseif obj:IsA("ImageLabel") or obj:IsA("ImageButton") then achado = obj.Image
    else
        for _, d in ipairs(obj:GetDescendants()) do
            if d:IsA("Decal") or d:IsA("Texture") then achado = d.Texture break end
            if d:IsA("ImageLabel") then achado = d.Image break end
        end
    end
    if achado and achado ~= "" then
        IMAGEM_FINAL = achado
        print("[Portal] ID convertido para imagem: " .. achado)
    end
end)

-- ========== TELA DE CREDITOS (12 segundos) ==========
local TIKTOK = "zinc.infinitx"
local DISCORD = "https://discord.gg/YQrHV4zFQ"
local TEMPO_TELA = 12

task.spawn(function()
    local gui = Instance.new("ScreenGui")
    gui.Name = "CreditosPortal"
    gui.ResetOnSpawn = false
    gui.DisplayOrder = 100
    gui.Parent = (gethui and gethui()) or game:GetService("CoreGui")

    local quadro = Instance.new("Frame", gui)
    quadro.Size = UDim2.new(0, 150, 0, 90)
    quadro.AnchorPoint = Vector2.new(1, 0)
    quadro.Position = UDim2.new(1, -8, 0, 56)
    quadro.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    Instance.new("UICorner", quadro).CornerRadius = UDim.new(0, 14)
    local borda = Instance.new("UIStroke", quadro)
    borda.Color = COR_PORTAL
    borda.Thickness = 1.5


    local function botao(y, rotulo, texto, copiar)
        local b = Instance.new("TextButton", quadro)
        b.Size = UDim2.new(1, -12, 0, 28)
        b.Position = UDim2.new(0, 6, 0, y)
        b.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
        b.Text = rotulo .. ": " .. texto
        b.Font = Enum.Font.GothamMedium
        b.TextScaled = true
        b.TextColor3 = Color3.new(1, 1, 1)
        Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
        b.MouseButton1Click:Connect(function()
            local ok = pcall(function() setclipboard(copiar) end)
            b.Text = ok and "Copiado!" or "Seu executor nao copia"
            task.delay(1.5, function() b.Text = rotulo .. ": " .. texto end)
        end)
    end
    botao(6, "TikTok", TIKTOK, TIKTOK)
    botao(38, "Discord", DISCORD, DISCORD)

    local dica = Instance.new("TextLabel", quadro)
    dica.Size = UDim2.new(1, -12, 0, 13)
    dica.Position = UDim2.new(0, 6, 1, -17)
    dica.BackgroundTransparency = 1
    dica.Font = Enum.Font.Gotham
    dica.TextScaled = true
    dica.TextColor3 = Color3.fromRGB(170, 170, 170)

    for i = TEMPO_TELA, 1, -1 do
        dica.Text = "Toque pra copiar - fecha em " .. i .. "s"
        task.wait(1)
    end
    gui:Destroy()
end)

-- ========== SOM AO ABRIR O PORTAL ==========
local ID_SOM = 87842207789771
local somPortal = Instance.new("Sound")
somPortal.Name = "SomPortal"
somPortal.SoundId = "rbxassetid://" .. ID_SOM
somPortal.Volume = 1
somPortal.Parent = game:GetService("SoundService")

local function tocarSom()
    pcall(function()
        somPortal.TimePosition = 0
        somPortal:Play()
    end)
end

task.delay(6, function()
    if not somPortal.IsLoaded then
        warn("[Portal] O som " .. ID_SOM .. " nao carregou (ID invalido ou nao liberado).")
    end
end)

local portais = { A = nil, B = nil }
local info = { A = nil, B = nil }  -- posicao/direcao de cada portal
local bloqueio = nil               -- portal onde voce acabou de sair (nao volta sozinho)
local teleportando = false
local proximo = "A"
local ultimoTP = 0

-- ========== ONDE/COMO O PORTAL FICA ==========
-- chao = em pe (virado pra voce) | parede = em pe | ar e teto = deitado, virado pra baixo
local function calcular(r)
    local n = r.Normal
    if n.Y > 0.7 then
        -- chao: portal em pe, virado pra voce
        local hrp = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
        local d = hrp and (hrp.Position - r.Position) or Vector3.new(0, 0, 1)
        d = Vector3.new(d.X, 0, d.Z)
        if d.Magnitude < 0.1 then d = Vector3.new(0, 0, 1) end
        local f = d.Unit
        return r.Position + Vector3.new(0, ALTURA / 2, 0), f, false, nil
    elseif n.Y < -0.7 then
        -- ar ou teto: portal deitado, virado pra baixo
        local cam = workspace.CurrentCamera
        local ref = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
        ref = (ref.Magnitude > 0.1) and ref.Unit or Vector3.new(0, 0, -1)
        local f = Vector3.new(0, -1, 0)
        return r.Position + f * 0.1, f, true, ref
    end
    -- parede: portal em pe, virado pra fora da parede
    local f = Vector3.new(n.X, 0, n.Z).Unit
    return r.Position + f * 0.3, f, false, nil
end

-- ========== PORTAL ==========
local function criarPortal(letra, centro, f, plano, ref)
    local cf
    if plano then
        -- deitado: eixo X = pra cima/baixo, eixo Y (altura do oval) = direcao horizontal
        local h = ref - f * ref:Dot(f)
        h = (h.Magnitude > 0.1) and h.Unit or Vector3.new(0, 0, -1)
        cf = CFrame.fromMatrix(centro, f, h, f:Cross(h))
    else
        cf = CFrame.lookAt(centro, centro + f) * CFrame.Angles(0, math.pi / 2, 0)
    end
    local modelo = Instance.new("Model")
    modelo.Name = "Portal" .. letra
    local C = Enum.PartType.Cylinder

    local function oval(tam, cor, mat, c0)
        local p = Instance.new("Part")
        p.Shape = C
        p.Size = tam
        p.Color = cor
        p.Material = mat
        p.Anchored, p.CanCollide, p.CanTouch, p.CanQuery = true, false, false, false
        p.CFrame = cf * c0
        p.Parent = modelo
        return p
    end

    -- Area de toque (invisivel e mais grossa, pro teleporte pegar sempre)
    local disco = oval(Vector3.new(0.8, ALTURA, LARGURA), BORDAS[1], Enum.Material.SmoothPlastic, CFrame.new())
    disco.Name = "Disco"
    disco.Transparency = 1
    disco.CanTouch = true

    -- Imagem do portal (as duas faces), com camadas que se mexem de leve
    local IMAGEM = IMAGEM_FINAL
    local meio = oval(Vector3.new(0.12, ALTURA, LARGURA), VERDE_MEIO, Enum.Material.SmoothPlastic, CFrame.new())
    meio.Transparency = 0.2 -- fundo verde ate a imagem carregar
    meio.Material = Enum.Material.Neon

    local camadas = {}
    for _, lado in ipairs({ Enum.NormalId.Right, Enum.NormalId.Left }) do
        local sg = Instance.new("SurfaceGui")
        sg.Face = lado
        sg.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
        sg.PixelsPerStud = PPS
        sg.LightInfluence = 0
        sg.Parent = meio

        local function nova(tam, transp)
            local i = Instance.new("ImageLabel")
            i.AnchorPoint = Vector2.new(0.5, 0.5)
            i.Position = UDim2.fromScale(0.5, 0.5)
            i.Size = UDim2.fromScale(tam, tam)
            i.BackgroundTransparency = 1
            i.BorderSizePixel = 0
            i.Image = IMAGEM
            i.ScaleType = Enum.ScaleType.Stretch
            i.ImageTransparency = transp
            i.ImageColor3 = Color3.new(1, 1, 1) -- sem filtro: a imagem ja e dourada
            i.Parent = sg
            return i
        end

        local base = nova(1, 0)       -- imagem principal
        local b = nova(0.8, 0.45)     -- camada que balanca (da a ideia de girar)
        local c = nova(0.55, 0.5)     -- camada menor, no sentido contrario
        local us = Instance.new("UIScale", base)
        camadas[#camadas + 1] = { base = base, b = b, c = c, us = us }

        task.spawn(function()
            for _ = 1, 50 do
                if base.IsLoaded then
                    meio.Transparency = 1
                    return
                end
                task.wait(0.1)
            end
            warn("[Portal] A imagem NAO carregou (ID invalido, moderada ou nao e imagem).")
        end)
    end

    -- Movimento leve do redemoinho
    local tempo = 0
    local con
    con = RunService.Heartbeat:Connect(function(dt)
        if not modelo.Parent then con:Disconnect() return end
        tempo += dt
        local e = 1 + 0.025 * math.sin(tempo * 3)
        for _, k in ipairs(camadas) do
            k.us.Scale = e
            k.base.Rotation = 2 * math.sin(tempo * 1.1)
            k.b.Rotation = 12 * math.sin(tempo * 1.4)
            k.c.Rotation = -16 * math.sin(tempo * 1.0 + 1)
            k.b.ImageTransparency = 0.4 + 0.15 * math.sin(tempo * 2)
            k.c.ImageTransparency = 0.45 + 0.15 * math.sin(tempo * 2.6 + 1)
        end
    end)

    -- Luz verde
    local luz = Instance.new("PointLight", disco)
    luz.Color = COR_PORTAL
    luz.Brightness = 2
    luz.Range = 18

    modelo:SetAttribute("Dir", f)
    modelo.Parent = workspace

    return modelo, cf
end


-- ========== TELEPORTE (por distancia: funciona de qualquer lugar do mapa) ==========
local function dentroDoPortal(inf, pos, margem)
    local p = inf.cf:PointToObjectSpace(pos)
    if math.abs(p.X) > 3 + margem then return false end
    local ny = p.Y / (ALTURA / 2 + margem)
    local nz = p.Z / (LARGURA / 2 + margem)
    return (ny * ny + nz * nz) <= 1
end

local function teleportar(de)
    local para = (de == "A") and "B" or "A"
    local char = lp.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local dest = info[para]
    if not hrp or not dest then return end

    teleportando = true
    task.spawn(function()
        local ok, err = pcall(function()
            local destino
            if dest.plano then
                destino = dest.centro + dest.dir * 3.5
            else
                destino = dest.centro + dest.dir * 4 - Vector3.new(0, 0.5, 0)
            end
            -- carrega o mapa la antes de ir (evita cair no vazio quando e muito longe)
            if (hrp.Position - destino).Magnitude > 300 then
                pcall(function() lp:RequestStreamAroundAsync(destino, 1) end)
            end

            local alvo = CFrame.lookAt(destino, destino + (dest.plano and dest.ref or dest.dir))
            char:PivotTo(alvo) -- leva o personagem e tudo que ele carrega
            ultimoTP = tick()
            bloqueio = para -- so volta a funcionar nesse portal depois que voce sair de perto dele

            hrp.AssemblyLinearVelocity = Vector3.zero

            -- confere se o jogo te puxou de volta (anti-teleporte do servidor)
            task.wait(1)
            if hrp.Parent then
                local dist = (hrp.Position - alvo.Position).Magnitude
                print(string.format("[Portal] 1s depois do teleporte voce esta a %.1f studs do destino", dist))
                if dist > 15 then
                    warn("[Portal] O JOGO TE PUXOU DE VOLTA: o servidor desfaz o teleporte.")
                    ultimoTP = tick() + 4 -- evita ficar indo e voltando
                end
            end
        end)
        if not ok then
            warn("[Portal] Erro no teleporte: " .. tostring(err))
        end
        teleportando = false
    end)
end

RunService.Heartbeat:Connect(function()
    if teleportando then return end
    local char = lp.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    for _, l in ipairs({ "A", "B" }) do
        local inf = info[l]
        if inf and portais[l] and portais[l].Parent then
            if bloqueio == l then
                -- libera de novo quando voce sai da area do portal
                if not dentroDoPortal(inf, hrp.Position, 2.5) then bloqueio = nil end
            elseif tick() - ultimoTP > COOLDOWN and dentroDoPortal(inf, hrp.Position, 0.5) then
                if info[l == "A" and "B" or "A"] then
                    teleportar(l)
                    return
                end
            end
        end
    end
end)

-- ========== CLIQUE / TOQUE ==========
local function acharPonto()
    local cam = workspace.CurrentCamera
    local ray = cam:ScreenPointToRay(mouse.X, mouse.Y)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    local ign = { lp.Character, portais.A, portais.B }
    for i = #ign, 1, -1 do if not ign[i] then table.remove(ign, i) end end
    params.FilterDescendantsInstances = ign
    return workspace:Raycast(ray.Origin, ray.Direction * ALCANCE, params), ray
end

local function colocar()
    local ok, err = pcall(function()
        local r, ray = acharPonto()
        if not r then
            -- nao clicou em nada (ceu): abre o portal no AR, deitado e virado pra baixo
            r = { Position = ray.Origin + ray.Direction * DIST_AR, Normal = Vector3.new(0, -1, 0) }
        end
        local l = proximo
        if portais[l] then portais[l]:Destroy() end
        local centro, f, plano, ref = calcular(r)
        local m, cf = criarPortal(l, centro, f, plano, ref)
        portais[l] = m
        info[l] = { cf = cf, centro = centro, dir = f, plano = plano, ref = ref }
        if bloqueio == l then bloqueio = nil end
        tocarSom()
        proximo = (l == "A") and "B" or "A"
    end)
    if not ok then
        warn("[Portal] Erro ao criar portal: " .. tostring(err))
    end
end

-- ========== ARMA FEITA A MAO (so usada se o ID da arma falhar) ==========
local function criarArmaPadrao()
local tool = Instance.new("Tool")
tool.Name = "Arma de Portal"
tool.RequiresHandle = true
tool.CanBeDropped = false

local ESCALA = 0.5 -- tamanho da arma
local BRANCO = Color3.fromRGB(240, 240, 240)
local CINZA = Color3.fromRGB(190, 190, 190)
local PRETO = Color3.fromRGB(25, 25, 25)
local VERDE = Color3.fromRGB(70, 255, 70)

local handle = Instance.new("Part")
handle.Name = "Handle"
handle.Size = Vector3.new(0.5, 1.5, 0.6) * ESCALA   -- cabo
handle.Color = BRANCO
handle.Material = Enum.Material.SmoothPlastic
handle.CanCollide = false
handle.Parent = tool

local function peca(forma, tam, c0, cor, mat, transp)
    local p = Instance.new("Part")
    p.Shape = forma
    p.Size = tam * ESCALA
    p.Color = cor
    p.Material = mat or Enum.Material.SmoothPlastic
    p.Transparency = transp or 0
    p.CanCollide, p.Massless = false, true
    local w = Instance.new("Weld")
    w.Part0, w.Part1 = handle, p
    w.C0 = CFrame.new(c0.Position * ESCALA) * (c0 - c0.Position)
    w.Parent = p
    p.Parent = tool
    return p
end

local BLOCO, BOLA, CIL = Enum.PartType.Block, Enum.PartType.Ball, Enum.PartType.Cylinder
local PE = CFrame.Angles(0, 0, math.pi / 2) -- deixa o cilindro em pe

-- corpo branco com pontas arredondadas
peca(BLOCO, Vector3.new(0.9, 0.55, 3.0), CFrame.new(0, 1.0, -1.0), BRANCO)
peca(CIL, Vector3.new(0.55, 0.9, 0.9), CFrame.new(0, 1.0, -2.5) * PE, BRANCO)  -- ponta da frente
peca(CIL, Vector3.new(0.55, 0.9, 0.9), CFrame.new(0, 1.0, 0.5) * PE, BRANCO)   -- ponta de tras
peca(BLOCO, Vector3.new(0.92, 0.1, 3.0), CFrame.new(0, 0.72, -1.0), CINZA)      -- faixa cinza embaixo

-- botao vermelho em cima
peca(BLOCO, Vector3.new(0.6, 0.06, 0.85), CFrame.new(0, 1.3, -1.0), Color3.fromRGB(120, 20, 25))
peca(BLOCO, Vector3.new(0.45, 0.08, 0.7), CFrame.new(0, 1.33, -1.0), Color3.fromRGB(215, 40, 45))

-- botao preto redondo dos dois lados
peca(CIL, Vector3.new(0.25, 0.5, 0.5), CFrame.new(0.5, 1.0, 0.1), PRETO)
peca(CIL, Vector3.new(0.25, 0.5, 0.5), CFrame.new(-0.5, 1.0, 0.1), PRETO)

-- ponta de baixo do cabo (argola/bulbo)
peca(BOLA, Vector3.new(0.7, 0.55, 0.85), CFrame.new(0, -0.9, 0.1), BRANCO)

-- capsula verde de vidro com energia dentro
peca(CIL, Vector3.new(0.12, 0.8, 0.8), CFrame.new(0, 1.34, -1.9) * PE, CINZA)  -- base
peca(CIL, Vector3.new(1.5, 0.7, 0.7), CFrame.new(0, 2.15, -1.9) * PE, Color3.fromRGB(130, 255, 130), Enum.Material.Glass, 0.45)
peca(BOLA, Vector3.new(0.7, 0.7, 0.7), CFrame.new(0, 2.9, -1.9), Color3.fromRGB(130, 255, 130), Enum.Material.Glass, 0.45)
local nucleo = peca(BOLA, Vector3.one * 0.5, CFrame.new(0, 2.2, -1.9), VERDE, Enum.Material.Neon)
peca(BOLA, Vector3.one * 0.28, CFrame.new(0, 2.2, -1.9), Color3.fromRGB(235, 255, 235), Enum.Material.Neon)

local luzArma = Instance.new("PointLight", nucleo)
luzArma.Color = VERDE
luzArma.Range = 6

-- faiscas de energia dentro da capsula
local fa = Instance.new("ParticleEmitter", nucleo)
fa.Color = ColorSequence.new(Color3.fromRGB(150, 255, 120), Color3.fromRGB(230, 255, 230))
fa.LightEmission = 1
fa.Rate = 25
fa.Lifetime = NumberRange.new(0.25, 0.5)
fa.Speed = NumberRange.new(0.5, 1.5)
fa.SpreadAngle = Vector2.new(360, 360)
fa.Size = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 0.2 * ESCALA * 2),
    NumberSequenceKeypoint.new(1, 0),
})
    return tool
end

-- ========== ARMA PELO ID (modelo) + SUA IMAGEM SO NO ICONE ==========
local ID_ARMA = 15484783837        -- modelo da arma que fica na mao
local ID_ICONE = 77773883701372    -- sua imagem (so aparece na barra de itens)
local TAMANHO_ARMA = 2.2           -- maior lado da arma, em studs
local GIRO = Vector3.new(0, 0, 0)  -- gira a arma na mao (graus X, Y, Z) se ficar torta

-- se o ID for de um Decal, pega o ID real da imagem dentro dele
local function resolverImagem(id)
    local imagem = "rbxassetid://" .. id
    pcall(function()
        local obj = game:GetObjects(imagem)[1]
        if not obj then return end
        local achado
        if obj:IsA("Decal") or obj:IsA("Texture") then achado = obj.Texture
        elseif obj:IsA("ImageLabel") then achado = obj.Image
        else
            for _, d in ipairs(obj:GetDescendants()) do
                if d:IsA("Decal") or d:IsA("Texture") then achado = d.Texture break end
                if d:IsA("ImageLabel") then achado = d.Image break end
            end
        end
        if achado and achado ~= "" then imagem = achado end
    end)
    return imagem
end

local JUNTAS = { Weld = true, WeldConstraint = true, ManualWeld = true, Motor6D = true, Motor = true, Snap = true }

local function carregarArma()
    local ok, objs = pcall(function() return game:GetObjects("rbxassetid://" .. ID_ARMA) end)
    local obj = ok and objs and objs[1]
    if not obj then
        warn("[Portal] Nao consegui carregar a arma " .. ID_ARMA .. ". Usando a arma padrao.")
        return nil
    end

    -- 1) junta todas as pecas do modelo
    local partes = {}
    if obj:IsA("BasePart") then
        partes[1] = obj
    else
        for _, d in ipairs(obj:GetDescendants()) do
            if d:IsA("BasePart") then partes[#partes + 1] = d end
        end
    end
    if #partes == 0 then
        warn("[Portal] O ID da arma nao tem nenhuma peca. Usando a arma padrao.")
        return nil
    end

    -- 2) escolhe a peca principal (Handle, ou a maior)
    local handle = obj:IsA("BasePart") and obj or nil
    if not handle then
        local h = obj:FindFirstChild("Handle", true)
        if h and h:IsA("BasePart") then handle = h end
    end
    if not handle then
        handle = partes[1]
        for _, p in ipairs(partes) do
            if p.Size.Magnitude > handle.Size.Magnitude then handle = p end
        end
    end

    -- 3) tamanho total da arma -> fator de escala
    local mn, mx = Vector3.one * math.huge, Vector3.one * -math.huge
    for _, p in ipairs(partes) do
        local m = p.Size / 2
        mn = Vector3.new(math.min(mn.X, p.Position.X - m.X), math.min(mn.Y, p.Position.Y - m.Y), math.min(mn.Z, p.Position.Z - m.Z))
        mx = Vector3.new(math.max(mx.X, p.Position.X + m.X), math.max(mx.Y, p.Position.Y + m.Y), math.max(mx.Z, p.Position.Z + m.Z))
    end
    local dim = mx - mn
    local maior = math.max(dim.X, dim.Y, dim.Z)
    local f = (maior > 0) and (TAMANHO_ARMA / maior) or 1

    -- 4) guarda a posicao de cada peca em relacao ao Handle
    local handleCF = handle.CFrame
    local rel = {}
    for _, p in ipairs(partes) do rel[p] = handleCF:ToObjectSpace(p.CFrame) end

    -- 5) apaga as juntas antigas
    if not obj:IsA("BasePart") then
        for _, d in ipairs(obj:GetDescendants()) do
            if JUNTAS[d.ClassName] then d:Destroy() end
        end
    end
    for _, p in ipairs(partes) do
        for _, d in ipairs(p:GetChildren()) do
            if JUNTAS[d.ClassName] then d:Destroy() end
        end
    end

    -- 6) monta uma Tool nova, tudo colado no Handle com o tamanho certo
    local tool = Instance.new("Tool")
    tool.Name = "Arma de Portal"
    tool.RequiresHandle = true
    tool.CanBeDropped = false

    for _, p in ipairs(partes) do
        p.Anchored = false
        p.CanCollide = false
        p.Massless = true
        p.Size = p.Size * f
        for _, d in ipairs(p:GetChildren()) do
            if d:IsA("Attachment") then d.Position = d.Position * f end
        end
        if p ~= handle then
            if p.Name == "Handle" then p.Name = "Peca" end
            local r = rel[p]
            local w = Instance.new("Weld")
            w.Part0, w.Part1 = handle, p
            w.C0 = CFrame.new(r.Position * f) * (r - r.Position)
            w.Parent = p
        end
    end
    handle.Name = "Handle"
    for _, p in ipairs(partes) do p.Parent = tool end

    tool.Grip = CFrame.Angles(math.rad(GIRO.X), math.rad(GIRO.Y), math.rad(GIRO.Z))
    print("[Portal] Arma carregada pelo ID " .. ID_ARMA .. " (" .. #partes .. " pecas)")
    return tool
end

-- ========== ARMA NUNCA SOME (volta sozinha depois de morrer) ==========
local molde = carregarArma() or criarArmaPadrao() -- copia guardada (nunca entra no inventario)
molde.TextureId = resolverImagem(ID_ICONE)        -- sua imagem como icone na barra de itens

local function temArma()
    local mochila = lp:FindFirstChild("Backpack")
    return (mochila and mochila:FindFirstChild("Arma de Portal"))
        or (lp.Character and lp.Character:FindFirstChild("Arma de Portal"))
end

local function darArma()
    if temArma() then return end
    local mochila = lp:FindFirstChild("Backpack")
    if not mochila then return end
    local nova = molde:Clone()
    nova.Activated:Connect(colocar)
    nova.Parent = mochila
end

darArma()

-- vigia a mochila: se a arma sair dela (morte, jogo limpando itens), volta na hora
local function vigiar(mochila)
    mochila.ChildRemoved:Connect(function(c)
        if c.Name == "Arma de Portal" then
            task.wait(0.2) -- se foi so equipada, ela ja estara na mao e nao duplica
            pcall(darArma)
        end
    end)
end

local mochilaAtual = lp:FindFirstChild("Backpack")
if mochilaAtual then vigiar(mochilaAtual) end

-- depois de morrer o jogo cria uma mochila nova: coloca a arma nela assim que aparecer
lp.ChildAdded:Connect(function(c)
    if c.Name == "Backpack" then
        vigiar(c)
        task.wait(0.1)
        pcall(darArma)
        task.wait(0.5)
        pcall(darArma)
    end
end)

lp.CharacterAdded:Connect(function()
    for _, t in ipairs({ 0.2, 0.7, 1.5, 3 }) do
        task.wait(t)
        pcall(darArma)
    end
end)

-- garantia extra: confere de novo a cada meio segundo
task.spawn(function()
    while true do
        task.wait(0.5)
        pcall(darArma)
    end
end)

print("Arma de Portal na mochila. Equipe e clique/toque onde quer o portal.")
