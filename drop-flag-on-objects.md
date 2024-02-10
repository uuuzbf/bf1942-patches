## Patch for CTF flags falling trough map objects
This patch fixes the issue that CTF flags always fall onto the map terrain, falling trough any buildings the player might stand on.

Flag on top of rubber pile:
![image](https://github.com/uuuzbf/bf1942-patches/assets/135877649/445c5e0a-759e-4082-b1d0-4410d9bead38)

## TL;DR

### Patching bf1942_lnxded.static
At offset 0x249E80 replace 152 (0x98) bytes, replace this:

```
D8 05 D0 E4 6B 08 58 8B 45 98 89 45 B8 5A 8B 45 9C 8B 55 08 89 45 BC 8B 45 A0 D9 45 AC D9 45 B0 89 45 C0 D9 45 C0 D9 45 BC D9 CC D9 5D DC D9 C2 D9 C2 D9 C9 D8 CA D9 45 A8 D9 CA D8 CE D9 45 B8 D9 CA DE E1 D9 C2 D9 CE D8 CA D9 CB D8 CF D9 C9 D9 55 C8 D9 CE D8 CC D9 C9 DE E3 D9 CC D8 C9 D9 C3 D8 CE D9 C9 DE E5 D9 C6 D9 CB D9 55 D0 DC CB D9 CD D9 55 CC D9 CD D8 CA D9 CC D8 CD D9 C9 DE E4 D9 C9 DE CC DE E9 D9 CC DE CB D9 5D AC DE E1 D9 C9 D9 5D A8 D9 5D B0
```

with these new bytes:

```
d9 54 24 04 c7 04 24 00 02 00 02 8b 75 0c 8b 06 56 ff 50 3c 89 04 24 68 38 bb 71 08 6a 00 6a 00 d8 63 34 d9 5c 24 00 6a 00 8d 43 30 ff 70 08 ff 70 04 ff 30 83 ec 20 8d 44 24 38 50 83 e8 0c 50 83 e8 0c 50 83 e8 04 50 83 e8 0c 50 83 e8 0c 50 83 e8 04 50 a1 24 dc 71 08 50 8b 08 ff 51 48 83 c4 20 84 c0 74 06 d9 44 24 08 eb 04 d9 44 24 44 d8 05 d0 e4 6b 08 d9 5d dc 83 c4 48 8b 55 08 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90
```


### Patching bf1942_lnxded.dynamic
At offset 0x250EF0 replace 152 (0x98) bytes, replace this:

```
D8 05 D0 27 67 08 58 8B 45 98 89 45 B8 5A 8B 45 9C 8B 55 08 89 45 BC 8B 45 A0 D9 45 AC D9 45 B0 89 45 C0 D9 45 C0 D9 45 BC D9 CC D9 5D DC D9 C2 D9 C2 D9 C9 D8 CA D9 45 A8 D9 CA D8 CE D9 45 B8 D9 CA DE E1 D9 C2 D9 CE D8 CA D9 CB D8 CF D9 C9 D9 55 C8 D9 CE D8 CC D9 C9 DE E3 D9 CC D8 C9 D9 C3 D8 CE D9 C9 DE E5 D9 C6 D9 CB D9 55 D0 DC CB D9 CD D9 55 CC D9 CD D8 CA D9 CC D8 CD D9 C9 DE E4 D9 C9 DE CC DE E9 D9 CC DE CB D9 5D AC DE E1 D9 C9 D9 5D A8 D9 5D B0
```

with these new bytes:

```
d9 54 24 04 c7 04 24 00 02 00 02 8b 75 0c 8b 06 56 ff 50 3c 89 04 24 68 38 96 6c 08 6a 00 6a 00 d8 63 34 d9 5c 24 00 6a 00 8d 43 30 ff 70 08 ff 70 04 ff 30 83 ec 20 8d 44 24 38 50 83 e8 0c 50 83 e8 0c 50 83 e8 04 50 83 e8 0c 50 83 e8 0c 50 83 e8 04 50 a1 24 b7 6c 08 50 8b 08 ff 51 48 83 c4 20 84 c0 74 06 d9 44 24 08 eb 04 d9 44 24 44 d8 05 d0 27 67 08 d9 5d dc 83 c4 48 8b 55 08 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90
```

## Patch details
This patch is applied inside `Flag::handleDrop`.
A look at the original function's pseudocode:
```c++
void Flag::handleDrop(Flag *this, BFPlayer *player, BFSoldier *playerObject) {
  IObject *playerVehicle;
  long double terrainHeight;
  Vec3 normal;
  BaseMatrix4_float mt;

  dice::ref2::world::Item::handleDrop(&this->item0, player, playerObject);
  player->is_tagged = 0;
  this->item0.base.base.base.base.base.base.base.vtbl.Item->enable(this);
  if ( player->alive == 1 )
  {
    // If handleDrop was called with an alive player it's assumed that the player captured the flag.
    dice::bf::game->__parent.vtbl->handleScore(dice::bf::game, player, SCORE_EVENT_FLAGCAPTURE, 0, -1, -1);
    dice::ref2::world::Flag::reSpawn(this);
  }
  else
  {
    playerVehicle = player->getVehicle();
    // copy player vehicle's matrix to mt
    qmemcpy(&mt, playerVehicle->getAbsoluteTransformation(), sizeof(mt));
    normal.x = 0.0;
    normal.y = 0.0;
    normal.z = 0.0;
    // use getHeightAndNormal to get the terrain height at x,z point
    // it additionally returns the normal vector into normal
    terrainHeight = terrainBase->getHeightAndNormal(mt.pos.x, mt.pos.z, &normal);
    // The following code is mostly for rotating the flag object to make it perpendicual to the ground.
    mt.b.x = normal.x;
    mt.b.y = normal.y;
    mt.b.z = normal.z;
    // This sets the flags position to be 1.5 units above the terrain. This is the only useful code here.
    mt.pos.y = terrainHeight + 1.5;
    // More rotation code
    mt.c.x = mt.a.y * normal.z - mt.a.z * normal.y;
    mt.c.z = mt.a.x * normal.y - mt.a.y * normal.x;
    mt.c.y = mt.a.z * normal.x - mt.a.x * normal.z;
    mt.a.y = normal.z * mt.c.x - mt.c.z * normal.x;
    mt.a.x = normal.y * mt.c.z - normal.z * mt.c.y;
    mt.a.z = mt.c.y * normal.x - mt.c.x * normal.y;
    // Finally set the updated matrix as the flag's new transformation matrix.
    this->setAbsoluteTransformation(&mt);
  }
}
```


There is enough space to apply the patch because the original code does some calculations to rotate the flag to be perpendicular to the terrain. However this code is useless because flag rotations are not synced to clients and do not affect flag behavior. A similiar rotation code is present in the Kit drop code, probably thats the source of it.

The patch adds an extra call to `ObjectManager::intersectLine()`. This function does line tests on world objects, so it can be used to test if there is any object between where the player dropped the flag and the terrain below. `intersectLine` can also accept an `ObjectPredicator`, this is used to discard certain objects from the search. An `ExplosionPredicator` is used (nothing to do with explosions, its just used in the explosion code too) which skips `BFSoldier`s, can test if some object flags are set and can skip an additional object. The additional object will be the player's current vehicle, so the flag won't get stuck in it.

Patch bytes for `bf1942_lnxded.dynamic`:
```c
// terrain_height is in st(0)
// stack offset 7C, 8 bytes garbage at top
// ebx is a pointer to the matrix
0xd9, 0x54, 0x24, 0x04, // fst dword [esp+4] ; store terrain_height on stack
// need to add a call here to objectManager->intersectLine(
//      IObject **hitobject, BaseVector3_float *intersectpos, BaseVector3_float *intersectnormal, int *a5,  <- outputs
//      BaseVector3_float *startpos, BaseVector3_float *direction_offset, ObjectPredicator *predicator);    <- inputs
// space needed for parameter variables: 4+12+12+4 + 12+12+8 = 64 bytes

// set up ExplosionPredicator, this predicator skips BFSoldiers, checks object flags, and
// can skip an additional specified object
0xc7, 0x04, 0x24, 0x00, 0x02, 0x00, 0x02,   // mov [esp],2000200h ; predicator.objectFlags = IS_ROOT|HAS_COLLISION
0x8B, 0x75, 0x0C,               // mov esi,[ebp+0Ch] ; get BFPlayer* from arg list
0x8B, 0x06,                     // mov eax, [esi] ; eax = BFPlayer vtable
0x56,                           // push esi ; BFPLayer* 
0xFF, 0x50, 0x3C,               // call eax+BFPlayer_vtbl.getVehicle] ; call BFPlayer->getVehicle()
0x89, 0x04, 0x24,               // mov [esp], eax ; predicator.ignoreObject = vehicle (replaces getVehicle argument on stack)
0x68, 0x38, 0x96, 0x6c, 0x08,   // push 086C9638 ; predicator.vtbl for .dynamic executable, static = 0871BB38

0x6a, 0x00,         // push 0 ; direction_offset.z = 0
// direction_offset.y = terrain_height - mt.pos.z
0x6a, 0x00,         // push 0
0xd8, 0x63, 0x34,   // fsub [ebx+34h]
0xd9, 0x5c, 0x24, 0x00, // ftsp dword [esp] ; terrain_height is no longer in fp register!
0x6a, 0x00,         // push 0 ; direction_offset.x = 0

0x8d, 0x43, 0x30,   // lea eax,[ebx+30h] ; eax = mt.pos ptr
0xff, 0x70, 0x08,   // push [eax+8] ; startpos.z = mt.pos.z
0xff, 0x70, 0x04,   // push [eax+4] ; startpos.y = mt pos.y
0xff, 0x30,         // push [eax] ; startpos.x = mt.pos.x

// rest of the variables are for output, dont need to initialize them (32 bytes)
0x83, 0xec, 0x20,   // sub esp,32

// push variable pointers to the stack (arguments for intersectLine)
0x8d, 0x44, 0x24, 0x38, // lea eax,[esp+38h] ; eax = predicator ptr
0x50,                   // push eax
0x83, 0xe8, 0x0c,       // sub eax,12 ; eax = direction_offset ptr
0x50,                   // push eax
0x83, 0xe8, 0x0c,       // sub eax,12 ; eax = startpos ptr
0x50,                   // push eax
0x83, 0xe8, 0x04,       // sub eax,4 ; eax = unknown ptr
0x50,                   // push eax
0x83, 0xe8, 0x0c,       // sub eax,12 ; eax = intersectnormal ptr
0x50,                   // push eax
0x83, 0xe8, 0x0c,       // sub eax,12 ; eax = intersectpos ptr
0x50,                   // push eax
0x83, 0xe8, 0x04,       // sub eax,4 ; eax = hitobject ptr
0x50,                   // push eax
// get object manager ptr, push this ptr and call intersectLine
0xa1, 0x24, 0xb7, 0x6c, 0x08,   // mov eax,[086CB724] // ecx = objectManager (dynamic, static = 0871DC24)
0x50,                   // push eax ; objectManager ptr
0x8b, 0x08,             // mov ecx,[eax]; vtable for ObjectManager
0xff, 0x51, 0x48,       // call [ecx+48] ; call objectManager->intersectLine()
0x83, 0xc4, 0x20,       // add esp,20h ; clean up stack, esp now points at our 64 bytes of arg objects again

// if an object was found by intersectLine, load intersectpos.y else load the saved terrain_height
0x84, 0xc0,             // test al,al
0x74, 0x06,             // jz no object found
// object was found:
0xD9, 0x44, 0x24, 0x08, // fld [esp+8] ; load intersectpos.y
0xeb, 0x04,             // jmp after next instruction
// object was not found:
0xD9, 0x44, 0x24, 0x44, // fld [esp+44h] ; load saved terrain_height from stack

// add 1.5 to final height and store it in mt.pos.y
0xd8, 0x05, 0xD0, 0x27, 0x67, 0x08, // fadd ds:const_flt_1_5 (add 1.5 to st(0), dynamic, static = 086BE4D0)
0xD9, 0x5D, 0xDC,                   // fstp [ebp+mt.pos.y]  ; store new height in position

// clean up stack
0x83, 0xc4, 0x48,   // add esp,70

// prepare call to setAbsoluteTransformation
0x8B, 0x55, 0x08,   // mov edx, [ebp+this] ; edx = this 

// 127 bytes used of 152 patched bytes, 25 nops
0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90,
0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90,
0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90,

// code continues at 08298F88 (dynamic, 08291F18 for static) 
```
For the `.static` executable 3 constants (marked in comments) need to be changed:
- Address of `ExplosionPredicator` virtual table
- Address of `objectManager` global
- Address of float const `1.5`
