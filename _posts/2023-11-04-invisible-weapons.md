---
layout: default
title: "invisible-weapons"
date: 2023-11-05 00:00:00 -0000
tags: picker vtmb noclip 
---

As mentioned in the last post, this one is going to go in depth on idiosyncrasies of the `picker` command. I'd be hard-pressed to call them bugs, since it was never intended to be used by players. The command will draw a bounding box around the entity you're looking at, and display some debug text. (Note: this command appears to be [generally available](https://developer.valvesoftware.com/wiki/List_of_Team_Fortress_2_console_commands_and_variables/en#picker) in Source engine games.) There are additional commands to toggle different categories of information. One of the idiosyncrasies will reveal an interesting implementation detail. 

I was exploring outside the playable area inside the player's apartment. The developers employ a common trick for "interior" maps when a window to the outside world is needed, which is to clone just enough of the outside to make a convincing illusion. I was looking around on this cloned chunk for any oddities, and found this:

![](/breaking-videogames/assets/sm_pawnshop_leveltrigger.png)

Why is a random level change trigger here? There's no corresponding trigger at the corresponding spot in the `sm_hub` map, as far as I can tell. There's also another oddity on this image that's worth dwelling on. Notice how the crosshair is nowhere near the highlighted entity.

The most straightforward, but not necessarily correct, explanation I could come up for this would be: the game sends out a ray towards where the crosshair is looking to determine what to highlight. Moreover, it would seem invisible entities such as level change triggers are excluded by default. 

The algorithm starts behaving strangely once you're outside the level bounds looking in, or looking towards the void. For example, you can sink under the map with `noclip` and highlight something above the point you're looking at. I'm not an expert on BSP rendering, but as far as I understand, everything outside the volume enclosed by the outermost walls are a sort of void for BSP-related algorithms. For example, you get the hall of mirrors effect because the renderer doesn't concern itself with anything outside the BSP volume.

It behaves as if the ray bounced upwards if it found itself out of bounds:

![Annotation made with over 9000 hours in MSPaint](/breaking-videogames/assets/sm_pawnshop_leveltrigger_annotated.png)

The exact behavior is not clear, because the highlight may go away depending on the view pitch. I have a potential explanation in mind, but I would need to brush up on some trigonometry before I could illustrate. You might have noticed another surprising consequence of this behavior: this lets you highlight entities you couldn't do by directly looking at them, such as level change triggers. This is what we're going to exploit.

Let's suppose you decide to get under the level and look down. Something like this is what will happen:

![](/breaking-videogames/assets/lookdown_noclip_picker.png)
![](/breaking-videogames/assets/baton_picker.png)

Depending on the angle you're looking down at, you'll see bounding boxes of various invisible weapons and items out in the void. At first I wasn't sure what all these items were, so I decided to try to move closer. To my surprise, the bounding box also moved. I assumed the renderer runs into some sort of undefined behavior trying to draw bounding boxes outside the map boundary, and these "phantom" items are actually hidden somewhere else on the map. But that's not what is happening.

Another clue for people familiar with the game: this is one of the items you might spot.

![](/breaking-videogames/assets/lockpicks_picker.png)

The real answer appears to be that what you're looking at is the contents of your inventory. Lockpicks are an item you pick up during the tutorial, and aren't available for pickup anywhere else in the game. It would appear that items in the player's inventory always float under/around the player, but are rendered invisible. However, the quirks of the raycast behavior mean that when you look down, you bounce the ray upwards, ignoring these entities' normal immunity to being highlighted.

Let me repeat, though: at all times, your character's entire inventory is invisibly floating behind/under you.

Bizarrely, you can even have the picker select the viewmodel of your current weapon:

![](/breaking-videogames/assets/viewmodel.png)


----

Maybe this would have been a satisfying conclusion, but it's possible to find out for sure. After writing the above, I did some more research trying to find documentation on the command's behavior. As it turns out, its source code is available in the Source SDK:

[Declaration of the "picker" console command](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/mp/src/game/server/baseentity.cpp#L5674)

The picker command toggles the `CBaseEntity::m_bInDebugSelect` variable, which is checked in `DrawAllDebugOverlays()` defined in `game/server/gameinterface.cpp`

[DrawAllDebugOverlays()](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/sp/src/game/server/gameinterface.cpp#L417)

The function responsible for selecting the entity to draw debug info for is:

```
CBaseEntity *FindPickerEntity( CBasePlayer *pPlayer )
```

Defined in [game/server/player.cpp](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/sp/src/game/server/player.cpp#L5788)

In turn it invokes two other functions, `CBaseEntity *FindEntityForward( CBasePlayer *pMe, bool fHull )` and failing that, `CBaseEntity *CGlobalEntityList::FindEntityNearestFacing( const Vector &origin, const Vector &facing, float threshold)`:

```C++
//-----------------------------------------------------------------------------
// Purpose: Finds the nearest entity in front of the player, preferring
//			collidable entities, but allows selection of enities that are
//			on the other side of walls or objects
// Input  :
// Output :
//-----------------------------------------------------------------------------
CBaseEntity *FindPickerEntity( CBasePlayer *pPlayer )
{
	MDLCACHE_CRITICAL_SECTION();

	// First try to trace a hull to an entity
	CBaseEntity *pEntity = FindEntityForward( pPlayer, true );

	// If that fails just look for the nearest facing entity
	if (!pEntity) 
	{
		Vector forward;
		Vector origin;
		pPlayer->EyeVectors( &forward );
		origin = pPlayer->WorldSpaceCenter();		
		pEntity = gEntList.FindEntityNearestFacing( origin, forward,0.95);
	}
	return pEntity;
}
```

Their respective definitions are here:

[FindEntityForward()](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/sp/src/game/server/player.cpp#L5726)
[FindEntityNearestFacing()](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/sp/src/game/server/entitylist.cpp#L1043)

I believe this explains the inconsistency with highlights:

```C++
//-----------------------------------------------------------------------------
// Purpose: Returns the nearest COLLIBALE entity in front of the player
//			that has a clear line of sight. If HULL is true, the trace will
//			hit the collision hull of entities. Otherwise, the trace will hit
//			hitboxes.
// Input  :
// Output :
//-----------------------------------------------------------------------------
CBaseEntity *FindEntityForward( CBasePlayer *pMe, bool fHull )
{
	if ( pMe )
	{
		trace_t tr;
		Vector forward;
		int mask;

		if( fHull )
		{
			mask = MASK_SOLID;
		}
		else
		{
			mask = MASK_SHOT;
		}

		pMe->EyeVectors( &forward );
		UTIL_TraceLine(pMe->EyePosition(),
			pMe->EyePosition() + forward * MAX_COORD_RANGE,
			mask, pMe, COLLISION_GROUP_NONE, &tr );
		if ( tr.fraction != 1.0 && tr.DidHitNonWorldEntity() )
		{
			return tr.m_pEnt;
		}
	}
	return NULL;

}
```

The `picker` command only cares about solids by default. So it's incorrect to say that the game will only highlight visible entities. Instead, it's going to prefer entities with collision. Failing that, it'll go through the global list of entities in the map, and after some filtering, tries to pick out the one whose position produces the smallest dot product with the player's facing:

```C++
//-----------------------------------------------------------------------------
// Purpose: Find the nearest entity along the facing direction from the given origin
//			within the angular threshold (ignores worldspawn)
// Input  : origin - 
//			facing - 
//			threshold - 
//-----------------------------------------------------------------------------
CBaseEntity *CGlobalEntityList::FindEntityNearestFacing( const Vector &origin, const Vector &facing, float threshold)
{
	float bestDot = threshold;
	CBaseEntity *best_ent = NULL;

	const CEntInfo *pInfo = FirstEntInfo();

	for ( ;pInfo; pInfo = pInfo->m_pNext )
	{
		CBaseEntity *ent = (CBaseEntity *)pInfo->m_pEntity;
		if ( !ent )
		{
			DevWarning( "NULL entity in global entity list!\n" );
			continue;
		}

		// Ignore logical entities
		if (!ent->edict())
			continue;

		// Make vector to entity
		Vector	to_ent = ent->WorldSpaceCenter() - origin;
		VectorNormalize(to_ent);

		float dot = DotProduct( facing, to_ent );
		if (dot <= bestDot) 
			continue;

		// Ignore if worldspawn
		if (!FStrEq( STRING(ent->m_iClassname), "worldspawn")  && !FStrEq( STRING(ent->m_iClassname), "soundent")) 
		{
			bestDot	= dot;
			best_ent = ent;
		}
	}
	return best_ent;
}
```
This function ignores whether an entity has collision. It's likely that entities like level change triggers are caught by this branch. It's unclear why it's only called once the player is outside the level bounds and/or looking into the void. I thought that it's because otherwise the ray hits the world and `FindEntityForward()` returns non-null, but this conditional seems to account for that:

```C++
if ( tr.fraction != 1.0 && tr.DidHitNonWorldEntity() )
```

[Declaration of DidHitNonWorldEntity()](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/sp/src/public/gametrace.h#L39)

My final guess is that it does fall through to `FindEntityNearestFacing()`, but in the engine version Vampire is running on, this conditional does not exist:

```C++
if (!FStrEq( STRING(ent->m_iClassname), "worldspawn")  && !FStrEq( STRING(ent->m_iClassname), "soundent")) 
```

Enabling picker mode will show worldspawn approximately all the time, which I thought was hardcoded behavior, but could just be this function consistently selecting worldspawn if nothing else is hit.

[Back to index](/breaking-videogames/)
