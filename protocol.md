# Refractor2 Vehicle Physics Research (BFH 1.78)

This document summarizes the reverse engineering research conducted on the Refractor2 vehicle physics system, with a focus on Battlefield Heroes 1.78 client.

The goal is to understand:

- How vehicle physics is structured internally
- Where velocity and force are applied
- Why direct velocity manipulation is unstable
- How real ramming behavior works
- How Havok is integrated

---

# 1. Physics Architecture Overview

Vehicles do NOT directly handle physics.

Runtime object hierarchy:

# Refractor2 Vehicle Physics Research (BFH 1.78)

This document summarizes the reverse engineering research conducted on the Refractor2 vehicle physics system, with a focus on Battlefield Heroes 1.78 client.

The goal is to understand:

- How vehicle physics is structured internally
- Where velocity and force are applied
- Why direct velocity manipulation is unstable
- How real ramming behavior works
- How Havok is integrated

---

# 1. Physics Architecture Overview

Vehicles do NOT directly handle physics.

Runtime object hierarchy:

Vehicle
└─ PhysicsOwner / Bundle
└─ PhysicsNode
└─ +0x10 → hkpRigidBody*

Critical pointer: vehicle_ptr + 0x4C → PhysicsNode*

Inside PhysicsNode: +0x10 → pointer to Havok hkpRigidBody


All authoritative physics state lives inside the Havok rigid body.

---

# 2. Linear Velocity Offsets (Confirmed)

Inside `PhysicsNode`:

| Offset | Type | Description |
|--------|------|-------------|
| +0x20 | float | Velocity X |
| +0x24 | float | Velocity Y |
| +0x28 | float | Velocity Z |

Confirmed via `sub_46FE0C`:

fmul
fstp [eax]
fstp [eax+4]
fstp [eax+8]


This is clearly a `Vector3` operation.

## Important Correction

These are NOT authoritative world velocity values.

They are:
- Integration buffers
- Havok input values
- Overwritten every simulation tick

Injecting velocity directly into these offsets causes:
- Side drift
- Jitter
- Overwrites next tick
- Solver instability

---

# 3. Sleepiness System

PhysicsNode relevant offsets:

| Offset | Type | Description |
|--------|------|-------------|
| +0x0C | BYTE | Active flag |
| +0x64 | int  | SleepinessMax |
| +0x68 | int  | CurrentSleepiness |

If `CurrentSleepiness == 0`:
- Physics update may be skipped
- Havok impulses may be ignored

Setting:

(BYTE)(physicsNode + 0x0C) = 1;


forces activity but does NOT override Havok motion type.

Sleepiness reset alone is insufficient for reliable movement.

---

# 4. Havok Integration (Critical Path)

Observed pattern: mov ecx, [ecx+10h] ; rigidBody
                  mov eax, [ecx] ; vtable
                  jmp [eax+90h] ; virtual call


VTable offset `+0x90` is highly likely: hkpRigidBody::applyLinearImpulse
                                        or
                                        hkpRigidBody::applyForce


## Key Insight

Real client ramming does NOT:
- Modify velocity buffer directly

Instead it:
- Computes direction vector
- Multiplies by boost strength
- Calls Havok ApplyImpulse

This is why tutorial ramming is stable.

---

# 5. MotionType Consideration

Havok impulse calls are ignored if rigid body motion type is:

- FIXED
- KEYFRAMED

Jeep rigid bodies are DYNAMIC, so impulse works correctly.

Velocity injection bypasses solver constraints and causes instability.

---

# 6. sub_7D84DB Clarification

Earlier assumption:
- sub_7D84DB globally locked objects to (0,0,0)

Updated understanding:
- Returns fallback zero vector
- Used in regulation systems
- Used in template fallback

It does NOT globally freeze vehicles.

---

# 7. Why Forward Boost Drift Happens

Example unstable hook:

```cpp
vel += forward * boost * dt;

Drift occurs because:

Forward vector may not be normalized

It may contain lateral components

Havok solver reprojects motion

Suspension + friction modify direction

Velocity buffer is not authoritative

Impulse-based force respects solver constraints.
Velocity injection does not.

Correct Boost Implementation Strategy

Correct method: physicsNode = *(vehicle + 0x4C)
                rigidBody   = *(physicsNode + 0x10)
                applyImpulse(rigidBody, direction * strength)

NOT:            vel[0] += ...

9. Updated PhysicsNode Layout
Offset	Type	Meaning
+0x0C	BYTE	Active flag
+0x10	PTR	hkpRigidBody*
+0x20	Vec3	Linear velocity buffer
+0x30	Vec3	Position buffer (non-authoritative)
+0x58	float	Gravity modifier
+0x64	int	SleepinessMax
+0x68	int	CurrentSleepiness
+0xEC	float	Vertical impulse accumulator

10. Summary of Findings

Incorrect assumptions corrected:

Velocity buffer ≠ true world velocity

Sleepiness reset alone ≠ guaranteed movement

sub_7D84DB ≠ global freeze function

Direct velocity injection ≠ stable boost

Confirmed truths:

Real ramming uses Havok impulse

rigidBody pointer at +0x10 verified

VTable +0x90 call path verified

Solver stability requires impulse-based force

Current Technical Position

Client Side:

Stable boost requires Havok impulse call

Velocity hacking is unstable

Direction vector extraction must be precise

Server Side:

No direct 0x707FB7 equivalent

Must replicate impulse physics manually

Requires locating server physics integration entry point








