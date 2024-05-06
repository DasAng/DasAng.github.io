+++
date = "2024-05-05T08:00:00-00:00"
title = "Total War: Warhammer 3 modding, create your own spell"
image = "/images/warhammer_mod.jpg"

+++

When I was a kid, I enjoyed collecting and painting Warhammer fantasy miniatures. I never imagined there would come a time when I could enjoy these tabletop games on a computer.

For this topic, let's shift our focus from the more serious matters to some fun and entertainment. I'll be demonstrating how to create a mod using the Lua programming language. This mod will enable us to expand the game by introducing new custom spells in unconventional ways, perhaps even surpassing the original intentions of the developers.

![](/images/great_unclean_one.jpg "Figure 1: Miniature of The Great Unclean One")

## Total War: Warhammer 3

If you're not yet acquainted with *Total War: Warhammer 3*, it's a turn-based strategy game featuring real-time tactical battles. Plenty of information about this game can be found online. However, this installment marks the final chapter of the trilogy *Total War: Warhammer*, and it introduces numerous additional races, with a special focus on the Daemons of Chaos, which happens to be my all-time favorite.

The essence of the game lies not only in commanding large armies to battle but also in unleashing devastating spells to obliterate the enemy into pieces.

![](/images/warhammer_spells.jpg "Figure 2: Casting spells to disrupt or destroy enemy units")

And let's not forget how visually stunning it is when you unleash a fire spell, reducing the enemy to ashes with its breathtaking visual effects.

## Mods, the way to keep the game alive

Since the release of *Total War: Warhammer 2*, I've been delving into the modding community, exploring what it takes to extend the game. It's a vast community with countless individuals creating mods, and some of them are truly mind-blowing.

I've always been captivated by the spellcasting aspect of the game. As a result, I set out to create my own spells, aiming not only to introduce fresh visual effects but also to fundamentally expand the game's functionality. Many of the mods and tutorials I've come across online, created by the modding community, primarily focus on enhancing visual effects or adjusting parameters such as damage output, range, and effects applied to units affected by spells etc.

![](/images/special_abilities_db.png "Figure 3: Snippet of how to configure various parts of a spell")

However, one aspect that seems lacking, or at least hasn't been explored extensively, is the ability to click and target any enemy unit on the battlefield, then apply specific game logic to that unit.

While the developers and the modding community have done an excellent job of exposing all the scripting functions that can be utilized in your own scripting mods, the game itself doesn't seem to handle events where you can point and click on an enemy unit, at least not in an obvious manner.

This is where I embarked on a quest to see if it was possible, and indeed, I have discovered a way to accomplish this.

## Let's create a spell to control enemy unit

So, the objective of this mod would be to create our own spell that introduces a new gameplay mechanic: the ability to control an enemy unit for a brief period, lasting about a minute or so. This would lay the foundation for doing all kinds of spell that can target enemy units.

You can download the complete mod file here, or you can follow along to understand the process step by step.

- [Download mod](https://github.com/DasAng/warhammer_mind_control_spell/releases/download/v1.0.0/ang_control_spell.pack)

You will need to have the [RPFM](https://github.com/Frodo45127/rpfm) installed to open the mod file and view it's content.

## Tools we need

For creating the mod we need the following tools and software:

- Total War: Warhammer 3
- [RPFM](https://github.com/Frodo45127/rpfm) for editing game database

And the modding community documentation for scripting:

- [Battle scripts documentation](https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_index.html)

## Start with the RPFM 

We will start by creating and empty mod file using **RPFM**. Open the RPFM application and create a new mod see figure 4.

![](/images/new_packfile.png "Figure 4: Choose New PackFile from menu to create a new mod")

This will leave us with an empty mod file. Next we will need to add add some database tables to the file. See figure 5

![](/images/rpfm_create_unit_special_ability_db.png "Figure 5: Steps to create a new database table for the new spell")

1. Create a new folder
2. Name the folder *db*
3. Create a new database table
4. Select the *unit_special_abilities* table and name it *ang_control_spell* or whatever name you like
5. The new *unit_special_abilities* database table should be created with empty rows.

## Editing tables

With the creation of our first database table, **unit_special_abilities**, we can now proceed to develop the additional database tables required to implement the basic functionality of our spell.

We will first add a new row to our **unit_special_abilities**, right click on the table and choose **"Add row"**. For a comprehensive listing of all column values, please refer to the [mod](https://github.com/DasAng/warhammer_mind_control_spell/releases/download/v1.0.0/ang_control_spell.pack). Here are the columns of interest:

| Column      | Value |
| -------------- | ----------- |
| Key            | mind_control |
| Mana Cost      | 12        |
| Num Uses      | -1 |

This table contains information pertaining to the spell, such as how much mana is required to cast the spell and how many times can it be used.

Next database table to add is **unit_abilities**. Add a new row to the table. Here are the columns of interest:

| Column      | Value |
| -------------- | ----------- |
| Key            | mind_control |
| Requires Effect Enabling      | false        |
| Icon name      | mind_control |
| Is Unit Upgrade | true |
| Source Type | spell |

This table contains information pertaining to the spell, such as which icon to display for it.

Last database table to add is **land_units_to_unit_abilites_junctions_tables**. Add a new row to the table. This table determines which character can use the spell.

| Column      | Value |
| -------------- | ----------- |
| Ability            | mind_control |
| Land Unit            | wh_main_chs_cha_archaon_the_everchosen_0 |

Here we will assign the spell to the character named *Archaeon the Everchosen*.

That's all the database tables we need for the spell to be enabled in the game.

## Icons and UI Text

The next step involves adding icons and text for the mind control spell to be displayed in-game. I'll leave it up to you to explore the [mod](https://github.com/DasAng/warhammer_mind_control_spell/releases/download/v1.0.0/ang_control_spell.pack) to see how it's implemented. It's quite straightforward, and I won't spend time explaining it here.

## Save Mod file

Go ahead and save the mod file and give it a name of your choice. Place it inside the Warhammer 3 data directory (usually in C:\Program Files (x86)\Steam\steamapps\common\Total War WARHAMMER III\data).

## Start the game

Now start the game and you should be able to see our spell in the army setup when you choose the **Chaos warriors** race and select **Archaeon The Everchosen** as your general.

![](/images/mind_control_spell.png "Figure 6: Shows the Mind Control spell we have added to the game")

![](/images/mind_control_spell_in_game.png "Figure 7: Display Archaeon The Everchosen is wielding the Mind Control spell in battle")

## Lua scripting

This is the crucial part that ensures the spell functions as intended. To enable the *Mind Control* spell to take control of an enemy unit, we'll need to provide some Lua code.

Our objective is outlined as follows:

1. The user selects the Mind Control spell from the spell panel to cast
2. We aim for the user to be able to click on any enemy unit within sight, excluding enemy generals, to gain control. (Controlling enemy generals would overly empower the spell.)
3. Upon gaining control, the enemy unit becomes responsive to our commands, allowing us to direct its actions.
4. The available commands include: 
    - move east
    - move south
    - move north
    - move west
    - halt
    - attack nearest enemy unit.
5. Control over the enemy unit is forfeited after one minute has elapsed.

Figure 8 shows a screenshot of the UI in game to control the enemy unit

![](/images/mind_control_ui_controls.png "Figure 8: Issue orders to enemy unit using the UI controls")

**Step 2** is pivotal for the successful execution of this spell. In the game mechanics, there's no inherent method to directly target an enemy unit. Spell functionalities typically revolve around modifying database parameters such as spell range or target selection criteria, rather than providing direct control over spells via Lua scripts.

### Make enemy unit clickable through UI

In order to provide a way for us to select an enemy unit on the battlefield I've discovered a workaround by leveraging the game function called [add_ping_icon()](https://chadvandy.github.io/tw_modding_resources/WH3/battle/script_unit.html#function:script_unit:add_ping_icon). This function essentially enables us to overlay a clickable UI icon onto any unit present on the battlefield, akin to the yellow "eye" icon depicted in figure 8.

When clicking on the icon, the game triggers events to the Lua script, signaling that a UI component has been clicked, allowing us to identify the targeted unit. However, this method bypasses the database parameter settings, such as range, as all control is managed exclusively through Lua scripts.

We might be stretching the function beyond its original design as intended by the game developers, but it's the only approach that provides us with the necessary outcome.

Now that we've established a method for identifying our enemy target via Lua scripting, let's delve into the specifics of the implementation.

### Script structure

To view all the Lua scripts, download the [mod](https://github.com/DasAng/warhammer_mind_control_spell/releases/download/v1.0.0/ang_control_spell.pack) and open it using RPFM. The scripts can be found inside the **script/battle/mod** folder. You can also look at the scripts in the [github repository](https://github.com/DasAng/warhammer_mind_control_spell).

The scripts are organized as follows:

- *ang_battle.lua*
- *ang_common.lua*
- *ang_mind_control_spell.lua*
- *ang_ui.lua*

**ang_battle.lua**. This script is the main entry point and is responsible for loading the *ang_mind_control_spell.lua* script.

**ang_common.lua**. This script contains common functionality being used by the *ang_mind_control_spell.lua* script

**ang_mind_control_spell.lua**. This script contains all the logic for the actual spell implementation.

**ang_ui.lua**. This script contains common UI related logic used by *ang_mind_control_spell.lua*.

### Implement the Mind Control spell

Let's delve into the **ang_mind_control_spell.lua** script, where the heart of the implementation resides.

```lua
-- variable initialization
local mindControlSpell = "mind_control"
local mindControlSpellImage = "mind_control.png"
local mindControlListener = "mind_control_spell"
local mindControlDurationSelection = 10000
MindControl = MindControl or {}

local mindControlSpellDuration = 60000
```

The provided code snippet merely initializes several variables for future use. Specifically, we aim for the spell to endure for 60 seconds, a duration dictated by the variable *mindControlSpellDuration*. Furthermore, *mindControlDurationSelection* is configured to 10 seconds, the allotted time for selecting an enemy unit.

#### Function to find the unique identifier for the clicked unit

In the subsequent segment of the code, we craft a function designed to furnish us with the unique identifier of the selected enemy unit. It's essential to note that each unit possesses its own distinct identifier.

```lua
--- @function getUnitUniqueId
--- @description This function will determine the unique id of a unit from a selected UI component.
--- It is used when an enemy unit is selected by clicking on the ping icon.
--- @uic The uicomponent that is being clicked on
--- @ending unique id of the unit that is extracted from the uicomponent path otherwise an empty string is returned.
--------------------------------------------------------------------------------------------------------------------------- 
local function getUnitUniqueId(uic)
	local uic_parent = uic;
	local name = uic_parent:Id();
	while name ~= "root" do
		if name == "modular_parent" then
			break;
		end;
		uic_parent = UIComponent(uic_parent:Parent());
		name = uic_parent:Id();
    end;
    if uic_parent then
        local unitId = uic_parent:Parent()
        if unitId then
            local uuid = UIComponent(unitId)
            return uuid:Id()
        end
    end
	return "";
end;
```

The process involves traversing the UI hierarchy of the clicked UI component in reverse order until we encounter a UI component named "modular_parent", thereby obtaining the unique identifier of the unit. We'll employ this function to identify the unit whenever the "ping icon" is clicked.


#### Create UI controls to issue orders

The next part of the code is where we create the UI controls to move around the enemy unit:

```lua
---------------------------------------------------------------------------------------------------------------------------
--- @function createMindControlButtons
--- @description This function creates the UI buttons for controlling enemy units under the
--- mind control spell
--------------------------------------------------------------------------------------------------------------------------- 
MindControl.createMindControlButtons = function()
    out("DEBUG: create mind control UI")
    local relativeComponent =
        find_uicomponent(
        core:get_ui_root(),
        "hud_battle",
        "winds_of_magic"
    )

    if relativeComponent then
        out("DEBUG: relative component: "..tostring(relativeComponent))
    end

    if not MindControl.moveEastButton then
        out("DEBUG: create mind control move east button")
        MindControl.moveEastButton = createButton(relativeComponent, "move_enemy_east", 42, 42, -140, 80, "ui/skins/default/icon_speed_controls_play.png", "CIRCULAR")
        MindControl.moveEastButton:SetTooltipText("Move enemy under the 'mind control spell' east", true)
        MindControl.moveEastButton:SetVisible(false)
        registerForClick(MindControl.moveEastButton, "move_enemy_east", MindControl.moveEnemyEast)
    end
    if not MindControl.haltButton then
        out("DEBUG: create mind control halt button")
        MindControl.haltButton = createButton(relativeComponent, "halt_enemy", 42, 42, -240, 130, "ui/skins/default/icon_halt.png")
        MindControl.haltButton:SetTooltipText("Halt enemy under the 'mind control spell'", true)
        MindControl.haltButton:SetVisible(false)
        registerForClick(MindControl.haltButton, "halt_enemy", MindControl.haltEnemy)
    end
    if not MindControl.attackButton then
        MindControl.attackButton = createButton(relativeComponent, "attack_enemy", 42, 42, -120, 130, "ui/skins/default/icon_melee.png", "CIRCULAR_TOGGLE")
        MindControl.attackButton:SetTooltipText("Instruct enemy under the 'mind control spell' to attack closest enemy unit", true)
        MindControl.attackButton:SetVisible(false)
        registerForClick(MindControl.attackButton, "attack_enemy", MindControl.attackEnemy)
    end

    ---  do the same for moving west, north and south
end

---------------------------------------------------------------------------------------------------------------------------
--- @function showMindControlButtons
--- @description This function shows or hides all the buttons for the spell "mind control"
--------------------------------------------------------------------------------------------------------------------------- 
MindControl.showMindControlButtons = function(visible)
    out("DEBUG: show UI: "..tostring(visible))
    if MindControl.moveEastButton then
       MindControl.moveEastButton:SetVisible(visible)
    end
    --- do the same for the remaining buttons
end
```

#### Move, Halt, Attack

Here is the code to move, halt and attack nearby enemy unit. We will be utilizing the built-in game functions *goto_location()*, *halt()* and *start_attack_closest_enemy()*.

```lua
---------------------------------------------------------------------------------------------------------------------------
--- @section move enemy east
---------------------------------------------------------------------------------------------------------------------------
MindControl.moveEnemyEast = function()
    out("DEBUG: move enemy east")
    if MindControl.selectedMindControlUnit then
        MindControl.selectedMindControlUnit:stop_attack_closest_enemy()
        local pos = MindControl.selectedMindControlUnit.unit:position()
        out("DEBUG: current unit position ("..tostring(pos:get_x())..","..tostring(pos:get_y())..","..tostring(pos:get_z())..")")
        local newpos = battle_vector:new(pos:get_x()+100,pos:get_y(),pos:get_z())
        out("DEBUG: new unit position ("..tostring(newpos:get_x())..","..tostring(newpos:get_y())..","..tostring(newpos:get_z())..")")
        MindControl.selectedMindControlUnit.uc:goto_location(newpos, true)
    end
end

--- do the same for the other direction: north, south and west

---------------------------------------------------------------------------------------------------------------------------
--- @section halt enemy unit to a stand still
---------------------------------------------------------------------------------------------------------------------------
MindControl.haltEnemy = function()
    out("DEBUG: halt enemy")
    if MindControl.selectedMindControlUnit then
        MindControl.selectedMindControlUnit:stop_attack_closest_enemy()
        MindControl.selectedMindControlUnit:halt()
    end
end

---------------------------------------------------------------------------------------------------------------------------
--- @section attack nearby enemy unit
---------------------------------------------------------------------------------------------------------------------------
MindControl.attackEnemy = function(context)
    out("DEBUG: attack closest enemy unit")
    if MindControl.selectedMindControlUnit then
        MindControl.selectedMindControlUnit:stop_attack_closest_enemy()
        MindControl.selectedMindControlUnit:start_attack_closest_enemy()
    end
end
```

These actions will be triggered whenever one of the UI control buttons is clicked.

#### Determine when the Mind Control spell is being cast

The following code segment is crucial for determining when our spell is being cast. Essentially, it entails comparing whether the UI component being clicked contains an icon named "mind_control.png". As you can imagine, we can apply this same logic to any custom spells we wish to implement in the future, by checking against the icon name of the custom spell.

```lua
--- @function getSpell
--- @description This function determines whether the passed in ui component is a spell button or not.
--- If it is a spell button then this function will return the name of the spell
--- @component the full image path for the spell. Fx: data/ui/Battle UI/ability_icons/mind_control.png
--------------------------------------------------------------------------------------------------------------------------- 
MindControl.getSpell = function(component)
    local componentImagePath = component:GetImagePath()
    local isspell = uicomponent_descended_from(component, "spell_parent")
    local currentState = component:CurrentState()
    out("DEBUG: image path "..tostring(component:GetImagePath()))
    out("DEBUG: is spell "..tostring(isspell))
    out("DEBUG: current state "..tostring(currentState))
    if isspell and currentState == "hover" and componentImagePath then
        out("DEBUG: check image path")
        if ends_with(componentImagePath, mindControlSpellImage) then
            out("DEBUG: got spell: "..tostring(mindControlSpell))
            return mindControlSpell
        end
    end
    return nil
end
```

#### Make enemy units selectable

This section of the code will display a yellow "ping icon" above each enemy unit, enabling them to be selected once we have cast the Mind Control spell. This utilizes the game function *add_ping_icon()*. This is the solution to enabling our custom spell to target any enemy unit through Lua scripting. By clicking on a UI component on the battlefield and determining the ID of the enemy unit from the clicked UI component, we've essentially enabled ourselves to apply various actions on the enemy unit dynamically.

```lua
---------------------------------------------------------------------------------------------------------------------------
--- @function activateMindControl
--- @description This function will activate the spell "mind control". It will find all enemy units
--- and put a ping icon on them so they are selectable. It will wait for the player to select one of the
--- enemy unit.
--------------------------------------------------------------------------------------------------------------------------- 
MindControl.activateMindControl = function()
    out("DEBUG: activate mind control spell")
    MindControl.clearMindControlUnits()
    MindControl.mindControlUnits = script_units:new("MindControlUnits")
    local armies = bm:get_non_player_alliance():armies()
    for i = 1, armies:count() do
        local army = armies:item(i)
        local units = army:units()
        for j = 1, units:count() do
            local unit = units:item(j)
            if unit and not unit:is_commanding_unit() then
                out("DEBUG: unit: "..tostring(unit:name()))
                local sunit = script_unit:new_by_reference(army, unit:name())
                sunit:add_ping_icon(2,mindControlDurationSelection)
                MindControl.mindControlUnits:add_sunits(sunit)
            end
        end
    end
    MindControl.lastSelectedMindControlUnitId = nil
end
```

This opens the door to many more potential custom spells in the future, allowing us to perform a variety of actions targeting enemy units through Lua scripting.

## Final thoughts

Crafting custom spells isn't overly difficult when the goal is to tweak various database parameters affecting the spells. However, delving into more advanced functionalities, diverging from the game's intended mechanics, poses a greater challenge.

Yet, with perseverance and resilience, I've demonstrated a method to circumvent the game mechanics, leveraging or perhaps pushing the boundaries of a game function to unlock exciting new functionalities for the game.