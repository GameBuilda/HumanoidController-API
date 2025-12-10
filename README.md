# HumanoidController-API
My second API (different from roblox's HumanoidController). Though doesn't target optimization, it targets users. Note, this wasn't tested :P except for some simple methods. Feedback is appreciated and helps ❤️.

DevForum : ...

1 :
What is a HumanoidController?
HumanoidControllers are lightweight, script-friendly wrapper around a Roblox Humanoid.

It standardizes and simplifies (or complicates?) :
-Damage
-Healing
-Teams and friendly-fire
-ForceField rules
-Status effects
-Signals and event flow
-Safe tracking (Mostly, damage sources without having to create creator tag instances.)
Note : Each Roblox Humanoid can only have one HumanoidController.
-> If you attempt to create another one for the same Humanoid, the module will simply return the existing controller.

What is a HumanoidSource?
HumanoidSources are basically tables that contain information about where the damage or heal came from.
It is used so you don't have to create creator tags that take several lines including debris. It contains
-Humanoid
-BaseParts (optional for effects like knockback)
-HumanoidController (not to be set but if there is a controller associated with the Humanoid, it gives this)
-Tag (an optional string to differentiate from other sources)

2 :
This module is intentionally not optimized for micro-performance.
It is designed to be:

Extremely clear
-Easy to extend
-Easy to debug
-Easy for teams of developers to read
-Easy to modify with custom behaviors
Submodules exist to avoid putting 1k+ lines in one module.
All public methods are also available through the main module, so developers only need to:
```lua
local HC = require(HumanoidController)
```

3 :
Getting Started
Creating a Controller
```
local HC = require(HumanoidController)
local controller = HC:GetController(Humanoid)
```
If the Humanoid doesn’t have a controller yet, it creates one. If it already has one, it returns the existing one.
Note : If the humanoid is invalid, it will error out by ```assert```

Creating a Source
```
local HC = require(HumanoidController)
local source = HC:NewSource(Humanoid, {BaseParts}?, "Some tag")
```

You can create a new source using the controller, this will automates setting the Humanoid and HumanoidController property.
```
local HC = require(HumanoidController)
local controller = HC:GetController(Humanoid)
local source = controller:GetSelfSource({BaseParts}?, "Some tag")
```

4 :
Core Concepts


1. Damage System - ```controller:TakeDamage()```
The damage system handles:
-Damage application
-Modification before taken using callback
-Friendly-fire logic (team system)
-ForceField rules (scroll deeper)
-Damage tweens
-Damage options
-Source tracking
-Damage signals (OnDamage)
-Last common method tracking

Before :TakeDamage() actually damages, it calls this callback
```
controller.OnTakingDamage = function(amount, source, options)
return amount * 0.5
end
```
This callback is optional and expects you to return a number as the final damage to be taken. This allows for changing damage on certain conditions without having to code it all.
-In the example, instead of taking 100% damage, it will take 50% damage instead.

OnDamage Signal
Fires on both Taking and Taken phases:
```
controller.OnDamage:Connect(function(amount, source, dtype)
    -- dtype = "Taking" before damage is taken or "Taken" after damage is taken
end)
```

DamageOptions
To create one, use
```
module:NewDamageOptions()
```
This will return a new table full of defaults which you can modify. This is used for further customization in ```controller:TakeDamage()``` as it takes a damage options as the 3rd parameter. (named Options meaning it is Optional)
The damage options include :
-Damage tweening (not TweenService) (modified using DamageTweenInfo) (goodbye creating loops!)
-Tween interruptions (includes damage tweens and healing. Any tweens that are currently ongoing will be stopped except this one. Can be excluded using DamageTweenInterruptBlackList and HealingTweenInterruptBlackList)
-Insta kill (instead of :TakeDamage(math.huge), you can just use the InstaKill boolean option to deal Humanoid.Health amount of damage.)
-etc..


2. Healing System - ```controller:GiveHeal()```
Healing is implemented with the identical structure of the damage system.
Not so fun fact : it is named *Give*Heal because it is the opposite of *Take*.
Includes :
-Healing application (obviously)
-Modification before taken using callback
-Healing tweens
-Healing options
-Source tracking
-Healing signals (OnHealing)
-Last common method tracking

Just like :TakeDamage(), it has an optional callback to modify the amount of heal before it is given :
```
controller.OnTakingHeal = function(amount, source, options)
return amount
end
```

OnHealing Signal
Fires on both Receiving and Received phases:
```
controller.OnHealing:Connect(function(amount, source, htype)
    -- htype = "Receiving" before heals is given or "Received" after helas is given
end)
```

HealingOptions
To create one, use
```
HC:NewHealingOptions()
```
Just like DamageOptions. This will return a new table full of defaults which you can modify. This is used for further customization in ```controller:GiveHeal()``` as it takes a healing options as the 3rd parameter.
The healing options include :
-Healing Tweens (modify using HealingTweenInfo)
-Insta heal (instead of :TakeDamage(math.huge), you can just use the InstaFullHeal boolean option to give Humanoid.Health amount of healing.)
-Shared heal (a table full of humanoid controllers. When :GiveHeal is called, it calculates whatever missing health is left then calls :GiveHeal to each one of them splitting the amount of the missing health as heal)


3. Teams and Friendly-Fire System
The team system provides a simple and fast approach to friendly-fire:
A HumanoidController can have a team name
A team can have allies
If friendly-fire is disabled, allies cannot damage each other.
If enabled, all damage flows normally.

If the team wasn't set for the controller and source, it will have the "Default" team which will always hurt anyone in this same team.
To set the team of a controller, use ```:SetTeam(teamName)``` where teamName is a string that represents the name of the team. If the team doesn't exist, then it will create a new one. Or you can do ```controller.Team = teamName``` which doesn't ensure a team exists and create one.
To set the team of a source, it must have a controller and then same directions as to how to set the team of the controller.
To add / remove an ally team of a team, use ```module.AddTeamAlly``` and ```module.RemoveTeamAlly``` which both takes a team name and team ally name to add / remove.

Key points :
“Friendly fire” simply means allowing damage between team members.
“Allies" are distinct teams that should not hurt each other.


4. ForceField Rules System
This system lets you define custom behaviors based on ForceField tags.

Example:
A “MagicShield” tag might absorb 50% damage, while “GodShield” absorbs all.
This system uses subtraction to bypass forcefields blocking damage and also uses CollectionService and Tags system.

Rules consist of:
Tag (identifier string on which forcefields to apply this rule to)
Priority (if theres a forcefield with multiple different rules, one will be picked by highest priority and that will be applied but if theres a forcefield with multiple different rules with the same priority, a random one will be picked. Same goes for multiple forcefields.)
AbsorbAmount
AbsorbMode (“Flat” or “Multiplier”) (Flat means the forcefield absorbs AbsorbAmount and the remaining damage will be taken, Multiplier means it will absorb Damage * Multiplier amount of damage. To completely nullify damage, set mode to Multiplier and set Multiplier to 0 or just set AllowDamage to true.)
AllowDamage (boolean that determnines if damage should pass. This works unless it is bypassed by using DamageOptions)
CustomData (unsure)

Note : These rules do not create the ForceField — they only define how an existing one behaves.
-> To create a forcefield that works with the rules, use :
```
module:CreateForceField(controller, tags {string})
```
This will add a forcefield to the controller with the tags to work with the rules. This will also return the created forcefield to add to debris or modify.

To add or remove an existing rule, do ```module.RegisterOrReplaceRule``` which takes a tag and a rule and ```module.RemoveFFRule``` which takes only a tag. But to register a rule, either put a table with working properties or for easier method is ```module.CreateNewRuleObject``` which returns a rule with defaults.

5. Status Effect System
Status effects are modular and decentralized.
Each effect is defined in /Effects as a module with methods :
```
function module:OnApply(runtime)
end

function module:OnRemove(runtime)
    -- reason = "Ended" or "Cleared"
end
```
Effects are not hardcoded into the main module — developers can add new ones simply by adding a new module inside /Effects to automatically register it.
To make it easier for developers to make effects, we have prepared a template inside the Effects submodule.

Params include: - ```module.CreateNewEffectParams()``` to create a new params that contains defaults.
Duration (how long the effect lasts for in seconds)
Source (optional, the humanoid source that is the root of this)

Options include: - ```module.CreateNewEffectOptions()``` to create a new option that contains defaults.
StackAmount
CustomData (unsure)
RefreshDuration (boolean that determines if after duration ends, it will instead remove 1 stack that has the same duration instead of ending the entire effect)

Effects automatically clear themselves when their duration finishes. Effects do not auto register unless they are in the Effects folder inside the Effects submodule.

To actually use these effects, use ```module.ApplyEffect(
	controller,
	effectName,
	params,
	options
)``` to apply an effect to a humanoid controller. This will fire a signal :
```
module.OnEffectApplied:Connect(function(controller, effectName, runtime)
end)
```
To remove a singular effect from a controller, use ```module.RemoveEffect(controller, effectName)``` that fires :
```
module.OnEffectRemoved:Connect(function(controller, effectName, runtime, reason : "Cleared" | "Ended")
print(reason) -- "Cleared"
end)
```
and to remove all effects, use ```module.ClearEffects(controller)```.
When an effect's duration ends, it will fire OnEffectRemoved with the reason "Ended".

To register an effect by script, do ```module.RegisterEffect(name, effectModule)```.

5:
Other Properties
Each humanoid controller exposes :
Humanoid : the humanoid associated with the controller
DefaultWalkSpeed : the walkspeed stored after the humanoid controller was created. Can be modified and extremely useful for getting the default speed meaning no more storing current speeds or using operation so much.
DefaultJumpPower : same as the default walkspeed but this time is for jump power.
Team : the string that determines what team the controller was on.
LastCommonMethod : "None" | "Damage" | "Heal" : determines what common method was last used.
LastSource : determines the last humanoid source that has to do something with the last common method used.

Other Methods
```controller:Destroy()``` - to prevent memory leakage, this is automatically called (but can be called manually) after the humanoid is removed (Parent = nil).

6:
End

Hopefully this was all enough to explain, but you could just understand by viewing its raw code, type definitions, method documentations, and by how the methods are named.
