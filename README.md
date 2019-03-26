# [Rift] Spell Database & Damage Library
This project is still under development and should be tested thoroughly when used inside of your scripts. If you notice any problems with the library please [File an Issue](https://github.com/tucker-rift/damage-library/issues/new) and I will get to it as soon as possible. 

This is a Spell Database and Damage (Prediction) Library developed by [DevTucker](https://forums.rift.lol/index.php?members/devtucker.1799/). Creator and developer of [PredatorAIO](https://discord.gg/AP9ptAD). The purpose of this library is to provide script developers with the utilities required to create better scripts by allowing them easy access to spell information and incoming damage calculation. Want to cast a life-saving spell at the last possible fraction of a second? This makes that easy. 

# Installation

The Spell Database & Damage Library is provided as a Dynamic Library which can easily be linked to your project. If you are not familiar with using libraries in C++ you can check out [this video](https://www.youtube.com/watch?v=pLy69V2F_8M). Once it's linked you simply need to include the respective `SpellDatabase.h` and `DamageLib.h` headers. The DamageLib relies on the SpellDatbase, so make sure to call `SpellDatabase::Init()` when you initialize your plugin. Failure to do this will cause crashes upon injection.

## Documentation

**DamageLib::CalculateDamage** returns type **std::pair<DamageType, float>**

Get the amount of damage that a spell will do to a given target using `DamageLib::CalculateDamage` which returns a `std::pair<DamageType, float>`

**Example (Printing damage your ultimate will do to each enemy)**: 
```c++
// Removing 10% from cast-range increases chances to hit with SDK Extensions prediction,
// from my experience :)
auto enemies = pSDK->EntityManager->GetEnemyHeroes(Ultimate.CastRange * 0.9);

for(auto &[netID, enemy] : enemies) 
{ 
    // Get the damage information for the ultimate "SpellSlot::R" 
    auto damageInfo = DamageLib::CalculateDamage(&Player, enemy, SpellSlot::R, SkillStage::Default);
    
    // Print the damage. 
    SdkUiConsoleWrite("Ultimate will do %f damage to %s", damageInfo.second, enemy->GetCharName());
}
```
```


```
---




**DamageLib::IsTargetKillable** returns type **bool**

This is a wrapper function for `DamageLib::CalculateDamage` and just returns true/false based on if the skill will kill the enemy if it hits. This takes Physical & Magical shield into consideration. 

**Example (Printing rather or not the target is killable)**: 
```c++
// Removing 10% from cast-range increases chances to hit with SDK Extensions prediction,
// from my experience :)
auto enemies = pSDK->EntityManager->GetEnemyHeroes(Ultimate.CastRange * 0.9);

for(auto &[netID, enemy] : enemies) 
{ 
    // Check to see if we can kill "enemy" will "SpellSlot::R"
    auto killable = DamageLib::IsTargetKillable(&Player, enemy, SpellSlot::R, SkillStage::Default);
    
    // Print the results. 
    SdkUiConsoleWrite("Ultimate will kill %s: %s", enemy->GetCharName(), killable ? "true" : "false");
}
```
```


```
---
**DamageLib::GetIncomingDamage** returns type **IncomingDamageReport**
Parameters `GetIncomingDamage(AIHeroClient* hero, float withinSeconds)`

This function calculates of all incoming damage within the next `n` seconds and provides an **IncomingDamageReport** which has the following information:

```c++
struct IncomingDamageReport
{ 
    DamageReport IncomingDamage;
    bool IsDeadly = false;  
    float DeathTime = 0.f;
};
```

The **DamageReport** is very simple and only contains the `Physical` and `Magical` properties which is the amount of that type of damage the hero is going to take, as-well as a `Total()` function which is just a utility for `Physical + Magical`. 

Using this information we can determine and react to death in several different ways, the example that I like to use is using Lux's W which is a skill-shot shield. We can detect if an allies *IncomingDamageReport* says that they're going to die and then get the calculated Time of death and determine if it's worth trying to shield them or not. 

There's a few reasons we may not want to shield them. 

 1. We can't shield them in time. Spells have a **CastDelay** which should be calculated against the **DeathTime** but skillshots such as Lux's W also have **Travel time** which is the amount of time it's going to take the shield to actually reach the player. So we need to do some logic such as `if(CurrentTime + CastDelay + TravelTime > DeathTime) DontCast`
 2. Shielding them is pointless, because our shield can't cover all of the incoming damage. 
 3. We hate them. 

**Example:**
```c++
// Cast W on a player if it will save their life. 
if(W->IsReady())
{
	auto allies = pSDK->EntityManager->GetAllyHeroes(W->Range - 40.f);
	for(auto &[_, player] : allies)
	{	
		// Dummy example not using proper collision timing, but good & simple enough for the example.
		// NOTE: Use prediction->CastPosition and not player position here. This is just for an example. 
		float timeToReachPlayer = Player.Distance(player) / W->Speed + W->Delay;

		// 0.25 seconds is the "Buffer" for calculation, this can be set to anything you want. I just liked 1/4th of a second. 
		float damageTimingThreshold = timeToReachPlayer + 0.25f;

		// Get the incoming damage report for the player. 
		auto damageReport = DamageLibrary::GetIncomingDamage(player, damageTimingThreshold);

		// If the player is not going to die, skip him.
		if (!damageReport.IsDeadly) continue;

		// If the player is going to die before the W projectile can reach & shield them, skip. 
		// There's no reason to waste your W cooldown. NOTE: Calculate ping here too ;) 
		if (damageReport.DeathTime < Game::Time() + timeToReachPlayer) continue;

		// If the amount of damage coming is deadly & Your shield can't cover the damage, skip.
		// There's no reason to waste your W cooldown. 
		if (damageReport.IncomingDamage.Total() > GetShieldAmount(SkillStage::Default)) continue;

		// Finally, cast the ability on the player. 
		// NOTE: Use prediction here, this is just for an example. 
		W->Cast(player);

		// We've casted, so break the function. 
		return; 
	}		
}
```


