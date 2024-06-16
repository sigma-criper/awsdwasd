-- Join discord: https://discord.gg/yFz3jEsp4b
-- Pretty sure this works, used Sirius esp lib to make. Not the greatest but you can see items, etc.
-- Will remake this with my own esp lib when i get time to make one for optimization.
-- // BYPASS
function errorLogSpoof()
 
    local old;old = hookmetamethod(game, "__namecall", function(self, ...)
        local args = {...}
        if getnamecallmethod() == "FireServer" and tostring(self) == 'UDPSocket' then
            if args[1]['script'] == 'LocalScript' then -- It's likely that the game knows their is no scripts named "LocalScript" so it bans. 
                return nil
            end
        end
        return old(self, ...)
    end)
end
 
errorLogSpoof()
-- // GLOBALS
 
getgenv().espObjects = espObjects or {
    ['Containers'] = {},
    ['Players'] = {},
    ['Zombies'] = {},
    ['Weapons'] = {},
    ['Medical'] = {},
    ['Edibles'] = {},
    ['Ammo'] = {},
    ['Attachments'] = {},
    ['Throwables'] = {},
    ['Clothing']  = {},
    ['Misc'] = {}
}
getgenv().connections = connections or {}
getgenv().plrTrackerObjects = plrTrackerObjects or Instance.new('Folder', gethui()) ; plrTrackerObjects.Name = '_Plrs'
 
do -- // CLEANUP
    for k, tbl in espObjects do
        for _, espObject in tbl do
            espObject:Destruct()
        end
    end
end
 
-- // VARIABLES
local player = game:GetService('Players').LocalPlayer
local espLibrary = loadstring(game:HttpGet('https://raw.githubusercontent.com/mrchigurh/Sirius/request/library/sense/source.lua'))()
 
local itemCFrameSpawnDirectory = game:GetService("ReplicatedStorage").Assets.ItemSpawnData.Items
local itemSpawnDirectory = game:GetService('Workspace').game_assets.item_spawns
local zombieSpawnDirectory = game:GetService('Workspace').game_assets.NPCs
local weaponData = game:GetService("ReplicatedStorage").GunData
 
local espData = {
    ['GroundItems'] = {
        ['Weapons'] = {
            Type = 'Instance',
            Contents = game:GetService("ReplicatedStorage").GunData,
        },
        ['Medical'] = {
            Type = 'Table',
            Contents = {
                'Bandage',
                'DressedBandage',
                'LargeMedkit',
                'Healing Salve',
                'Medkit',
                'LegSplit',
                'SalineSolution'
            },
        },
        ['Edibles'] = {
            Type = 'TableMatch',
            Contents = {
                'Apple',
                'Beans',
                'Juice',
                'Cereal',
                'Chips',
                'Cola',
                'Drink',
                'Potato',
                'Water',
                'DirtyWater',
                'EnergyDrink',
            }
        },
        ['Clothing'] = {
            Type = 'TableMatch',
            Contents = {
                'Shirt',
                'Jeans',
                'Slacks',
                'Suit',
                'Pants',
                'Jacket',
                'Sneakers',
                'Shoes',
                'Hoodie',
                'Backpack'
            }
        },
        ['Throwables'] = {
            Type = 'TableMatch',
            Contents = {
                'Molotov',
                'Frag',
            }
        },
        ['Ammo'] = {
            Type = 'ApartOfName',
            Contents = 'Ammo'
        },
        ['Misc'] = {
            Type = 'Default',
            Contents = {}
        },
    },
    ['Entities'] = {
        ['Players'] = {
            Type = 'Custom',
            Contents = function(instance)
                if instance.Parent == game.Workspace and table.find(game.Players:GetPlayers(), instance.Name) then
                    return 'Players'
                end
            end,
            CustomOptions = {
                boxOutline = false,
                textSize = 22,
            }
        },
        ['Zombies'] = {
            Type = 'Custom',
            Contents = function(instance)
                if instance.Parent == zombieSpawnDirectory then
                    return 'Zombies'
                end
            end
        },
    }
}
local espDirectories = {
    [itemSpawnDirectory.Name] = {
        Directory = itemSpawnDirectory,
        Exclude = {}
    },
    [itemCFrameSpawnDirectory.Name] = {
        Directory = itemCFrameSpawnDirectory,
        Exclude = { 'WorldModel' }
    },
    [zombieSpawnDirectory.Name] = {
        Directory = zombieSpawnDirectory,
        Exclude = {}
    },
}
local farAwayRenderLoot = { 'Weapons' }
 
-- // FUNCTIONS
 
do -- // ESP
    function initEsp() -- // SETS UP ALL ESP
 
        local function setupPlayerPseudoPart(plr)
            local pseudoPart = Instance.new('Part') ; pseudoPart.Parent = plrTrackerObjects
            pseudoPart.Name = plr.Name
            pseudoPart.Position = plr:GetAttribute('CharacterPosition')
            pseudoPart.Anchored = true
            pseudoPart.CanCollide = false
            pseudoPart.CanQuery = false
            pseudoPart.CanTouch = false
 
            createEspHandler(pseudoPart, 'Players')
 
            plr:GetAttributeChangedSignal('CharacterPosition'):Connect(function()
                local pos = plr:GetAttribute('CharacterPosition') + Vector3.new(0,3,0)
                pseudoPart.CFrame = CFrame.new(pos) --apparently more performant then .Position manually for frame time.
            end)
        end
 
        -- // Set up directories
 
        for key, directoryData in espDirectories do
 
            directoryData.Directory.ChildAdded:Connect(function(child)
                if table.find(directoryData.Exclude, child.Name) == nil then
                    createEspHandler(child)
                end
            end)
 
            for _, ins in directoryData.Directory:GetChildren() do
                if table.find(directoryData.Exclude, ins.Name) then continue end
                createEspHandler(ins)
            end
 
            warn(key, ' Set up...')
        end
 
        -- // Players
 
        for _, plr in game:GetService('Players'):GetPlayers() do
            if plr.Name == game:GetService('Players').LocalPlayer.Name then continue end
            setupPlayerPseudoPart(plr)
        end
 
        connections['pxAdded'] = game:GetService('Players').PlayerAdded:Connect(function(plr)
            if plr.Name == game.Players.LocalPlayer.Name then return end
            setupPlayerPseudoPart(plr)
        end)
 
        connections['pxLeft'] = game:GetService('Players').PlayerRemoving:Connect(function(plr)
            if plrTrackerObjects[plr.Name] then
                plrTrackerObjects[plr.Name]:Destroy()
            end
        end)
 
    end
 
    function getAssociation(instance)
        local itemName = instance.Name:gsub("%.item", "")
        local selectedAssocation = nil
        local customOptions = {}
        for categoryKey, mainInfo in espData do
            for key, itemInfo in mainInfo do
 
                if itemInfo['Type'] == 'Custom' then
                    local result = itemInfo['Contents'](instance)
                    if result ~= nil then
                        selectedAssocation = result
                        customOptions = itemInfo['CustomOptions'] or {}
                    end
                elseif itemInfo['Type'] == 'Instance' then
                    for _, ins in itemInfo['Contents']:GetChildren() do
                        if ins.Name == itemName then selectedAssocation = key end
                    end
                elseif itemInfo['Type'] == 'Table' then
                    for _, ins in itemInfo['Contents'] do
                        if string.lower(ins) == string.lower(itemName) then selectedAssocation = key end
                    end
                elseif itemInfo['Type'] == 'TableMatch' then
                    for _, ins in itemInfo['Contents'] do
                        if string.find(itemName, ins) ~= nil then selectedAssocation = key end
                    end
                elseif itemInfo['Type'] == 'ApartOfName' then
                    if string.find(itemName, itemInfo['Contents'], 1, false) ~= nil then selectedAssocation = key end
                end
 
            end
        end
 
        if selectedAssocation == nil then selectedAssocation = 'Misc' end
        return {[1] = selectedAssocation, [2] = customOptions}
    end
 
    function createEspHandler(instance, association)
        if instance:IsA('Folder') then return end
        local itemName = instance.Name:gsub("%.item", "")
        local itemInstance = nil
        local primaryPart = nil
        local associationInfo = getAssociation(instance)
        local customOptions = associationInfo[2] or {}
        association = association or associationInfo[1]
 
        if instance.Parent == itemCFrameSpawnDirectory then if table.find(farAwayRenderLoot, association) == nil then return end end
 
        if instance:IsA('Model') then
            primaryPart = instance.PrimaryPart
        else
            primaryPart = instance
        end
 
        itemInstance = primaryPart
        --itemInstance.Name = itemName
 
        -- // edge cases
 
        if instance.Parent == zombieSpawnDirectory then
            itemInstance.Name = 'Zombie'
        end
 
        local isEnabled = nil
        if Toggles['EspEnabled'].Value == false then
            isEnabled = false
        else
            isEnabled = Toggles[association].Value
        end
 
        local newEspObject = espLibrary.AddInstance(itemInstance, {
            enabled = isEnabled,
            text = itemName .. ' | {distance}', -- Placeholders: {name}, {distance}, {position}
            textColor = { Options[association..'Color'].Value, 1 },
            textOutline = true,
            textOutlineColor = Color3.new(),
            textSize = 12,
            textFont = 2,
            limitDistance = true,
            maxDistance = 3000,
        })
 
        table.insert(espObjects[association], newEspObject)
 
        if customOptions then
            for key, value in customOptions do
                print(key,value)
                customEspValueUpdate(association, key, value)
            end
        end
 
        instance.Destroying:Connect(function()
            if instance.Parent == nil or instance == nil then
                table.remove(espObjects[association], table.find(espObjects[association], newEspObject))
                newEspObject:Destruct()
            end
        end)
 
    end
 
    function customEspValueUpdate(category, customOption, customValue)
        if espObjects[category] == nil then warn(`{category} doesnt exist.`) ; return end
 
        if customOption ~= nil then
            for _, espObject in espObjects[category] do
                espObject.options[customOption] = customValue
            end
        end
    end
 
    function toggleCategoryEsp(category, bool)
        if espObjects[category] == nil then warn(`{category} doesnt exist.`) ; return end
 
        for _, espObject in espObjects[category] do
            espObject.options.enabled = bool
        end
    end
 
    function toggleEsp(bool)
        for categoryKey, espObjHolder in espObjects do
            if Toggles[categoryKey] == nil then continue end
            if Toggles[categoryKey].Value == false and bool == true then continue end
 
            toggleCategoryEsp(categoryKey, bool)
        end
    end
end
 
do -- // PLAYER
 
    function getClosestPlayer()
        local closest, closestMag = nil, math.huge
        for _, plr in game:GetService('Players'):GetPlayers() do
            if plr.Name == player.Name then continue end
            local mag = (plr:GetAttribute('CharacterPosition') - player.Character.WorldCharacter.PrimaryPart.Position).Magnitude
            if mag < closestMag then
                closest = plr
                closestMag = mag
            end
        end
        return closest
    end
 
end
 
do -- MISC INIT
 
    function initEspCache()
        for _, data in espData do
            for category, lk in data do
                if espObjects[category] == nil then
                    espObjects[category] = {}
                end
            end
        end
    end
 
end
 
-- // GUI
 
local repo = 'https://raw.githubusercontent.com/mrchigurh/LinoriaLib/main/'
 
local Library = loadstring(game:HttpGet(repo .. 'Library.lua'))()
local ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
local SaveManager = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()
local Window = Library:CreateWindow({
    Title = 'Aftermath GUI',
    Center = true,
    AutoShow = true,
})
 
local Tabs = {
 
    ['Main'] = Window:AddTab('Main'),
    ['ESP'] = Window:AddTab('ESP'),
    ['ESP Settings'] = Window:AddTab('ESP Settings'),
    ['UI Settings'] = Window:AddTab('UI Settings')
 
}
local EspGroupboxes = {}
local EspSettingsGroupboxes = {}
 
-- // ESP CREATION
 
Tabs['ESP Settings']:AddLeftGroupbox('GeneralEspSettings'):AddToggle('EspEnabled', {
    Text = 'ESP Toggle',
    Default = false,
    Tooltip = `Enable and disable all ESP.`,
    Callback = function(bool)
        toggleEsp(bool)
    end
}):AddKeyPicker('EspKeyPicker', {
    Default = 'E',
    SyncToggleState = true,
    Mode = 'Toggle',
 
    Text = 'Enable and disable esp',
    NoUI = false,
    Callback = function(Value)
        toggleEsp(Value)
    end,
    -- Occurs when the keybind itself is changed, `New` is a KeyCode Enum OR a UserInputType Enum
    ChangedCallback = function(New)
 
    end
})
 
 
 
for windowKey, generalData in espData do -- // ESP TAB
    EspGroupboxes[windowKey] = Tabs.ESP:AddLeftGroupbox(windowKey)
    EspSettingsGroupboxes[windowKey] = Tabs['ESP Settings']:AddLeftGroupbox(windowKey .. ` Distance Settings`)
 
    for itemKey, itemInfo in generalData do
 
        do --// ESP TOGGLES
            EspGroupboxes[windowKey]:AddToggle(itemKey, {
                Text = itemKey,
                Default = false,
                Tooltip = `All {itemKey} in the workspace.`,
                Callback = function(bool)
                    toggleCategoryEsp(itemKey, bool)
                end
            }):AddColorPicker(itemKey..'Color', {
                Default = Color3.new(0.419607, 0.788235, 0.529411),
                Title = `{itemKey} Color`, -- Optional. Allows you to have a custom color picker title (when you open it)
                Transparency = 0, -- Optional. Enables transparency changing for this color picker (leave as nil to disable)
 
                Callback = function(Value)
                    customEspValueUpdate(itemKey, 'textColor', {Value, 1})
                end
            })
        end
 
        do --// ESP DISTANCES
            EspSettingsGroupboxes[windowKey]:AddSlider(itemKey..'Slider', {
                Text = itemKey,
                Default = 3000,
                Min = 50,
                Max = 10000,
                Rounding = 1,
                Compact = true,
 
                Callback = function(Value)
                   customEspValueUpdate(itemKey, 'maxDistance', Value)
                end
            })
        end
    end
end
 
-- // MAIN UI
 
-- // UI OTHER
 
Library:OnUnload(function()
    print('Unloaded!')
 
    for _, part in plrTrackerObjects:GetChildren() do
        part:Destroy()
    end
 
    for k, tbl in espObjects do
        for _, espObject in tbl do
            espObject:Destruct()
        end
    end
 
    Library.Unloaded = true
end)
 
local MenuGroup = Tabs['UI Settings']:AddLeftGroupbox('Menu')
MenuGroup:AddButton('Unload', function() Library:Unload() end)
MenuGroup:AddLabel('Menu bind'):AddKeyPicker('MenuKeybind', { Default = 'LeftAlt', NoUI = true, Text = 'Menu keybind' })
 
-- SaveManager (Allows you to have a configuration system)
-- ThemeManager (Allows you to have a menu theme system)
 
ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ 'MenuKeybind' })
ThemeManager:SetFolder('linoria_lib')
SaveManager:SetFolder('linoria_lib/Aftermath')
SaveManager:BuildConfigSection(Tabs['UI Settings'])
ThemeManager:ApplyToTab(Tabs['UI Settings'])
SaveManager:LoadAutoloadConfig()
Library.ToggleKeybind = Options.MenuKeybind -- Allows you to have a custom keybind for the menu
 
-- // MAIN
 
--errorLogSpoof()
initEspCache()
initEsp()
