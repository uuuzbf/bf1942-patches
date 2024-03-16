## Patch for CTF flag respawn counter not resetting on pickup
 CTF flags have a 30 second respawn time which is counting down when the flag is on
 the ground. This counter is not reset when another player picks up or drops the flag,
 so the second time players have less time to collect the flag again. This patch fixes
 this by resetting the respawn counter on each pickup.

### Patching bf1942_lnxded.static
At offset `0x249DCD` replace 28 bytes, replace this:
```
50 56 E8 EC 0C DC FF 5A 59 50 A1 18 D9 70 08 50 FF 53 68 83 C4 20 EB A8 90 8D 76 00
```
with these bytes:
```
56 50 ff 53 68 8b 5d 08 8b 43 4c 8b 80 70 01 00 00 89 83 1c 01 00 00 83 c4 20 eb a4
```

### Patching bf1942_lnxded.dynamic
At offset `0x250E3D` replace 28 bytes, replace this:
```
50 56 E8 EC 0C DC FF 5A 59 50 A1 18 B4 6B 08 50 FF 53 68 83 C4 20 EB A8 90 8D 76 00
```

with these bytes:
```
56 50 ff 53 68 8b 5d 08 8b 43 4c 8b 80 70 01 00 00 89 83 1c 01 00 00 83 c4 20 eb a4
```


*New bytes are the same for both static and dynamic executables*

### Patch details
The following code is applied at `Flag::handlePickup` offset `0x5D`. There are some calls to `getBFPlayer` in the function, which doesnt do anything except returning its parameter. The last ˙getBFPlayer˙ call is removed to get some space and 4 bytes are used up from the free space after the function.
```asm
; remove a useless call to getBFPlayer to get some free bytes
push    esi ; flip this
push    eax ; and this instruction
; call   _ZN4dice2bf11getBFPlayerEPNS_4ref25world7IPlayerE
; pop    edx
; pop    ecx
; push   eax
; mov    eax,ds:_ZN4dice2bf4gameE
; push   eax
;  we now have 14 extra bytes!
call    DWORD PTR [ebx+0x68] ; (original) call pGame->handleScore()

; extra instructions
mov     ebx,DWORD PTR [ebp+0x8] ; ebx = Flag* this
mov     eax,[ebx+0x4c]  ; get Flag template ptr
mov     eax,[eax+0x170] ; get respawnTime from template
mov     [ebx+0x11c],eax ; set flag respawnCounter

add    esp,0x20
jmp    _ZN4dice4ref25world4Flag12handlePickupEPNS1_7IPlayerEPNS1_13IPlayerObjectE+0x1d
; 4 extra free bytes are also consumed after the original function
```
