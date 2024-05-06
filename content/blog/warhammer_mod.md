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

Now that we've established a method for identifying our enemy target via Lua scripting, let's delve into the specifics of the implementation.

