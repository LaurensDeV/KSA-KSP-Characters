# Using `kerbal_names.txt`

Notes for whoever writes the code mod. Checked against KSA `2026.10.19.5638` and
StarMap `0.4.8`; nothing in this repo reads the file yet.

## The file

One `<pool> <name>` per line, `#` comments, sorted and deduplicated per pool.

| pool | count | |
|---|---|---|
| `veteran` | 4 | Jebediah, Bill, Bob, Valentina |
| `special-m` | 79 | real-world and dev references |
| `special-f` | 56 | |
| `common-m` | 12800 | the generated pool |
| `common-f` | 12171 | |

## What KSP does with them

`CrewGenerator.GetRandomName` picks a gender, rolls `1` in `specialNameChance` (20) for
the special pool and takes the common pool otherwise, draws uniformly, then appends the
surname:

```csharp
pool = rng.Next(20) == 0 ? special[g] : common[g];
name = pool[rng.Next(pool.Length)] + " " + "Kerman";   // CrewGenerator.GetFullName
```

The four veterans are never generated — `KerbalRoster.GenerateInitialCrewRoster` hardcodes
them. Handing them out before anything else, once each, is the closest match to KSP.

## Where it goes in KSA

Two things make this less obvious than it looks.

**Do not patch `KittenNameGenerator.Generate`.** The character is not known there. Both call
sites pick the name and the character as two independent arguments:

```csharp
// KittenRosterData.CreateKitten
new KittenRosterEntryData(GenerateUniqueName(hashSet), PickRandomCharacterId());
```

Patching `Generate` renames the kittens too.

**Only rename entries that are unassigned and alive.** `EVADoor.CreateKittenEva` builds the
EVA vehicle with `id: rosterEntry.Name`, and `Universe.AssignStartingCrew` finds it back with
`CurrentSystem.All.TryGet(kitten.Name, out KittenEva _)`. Renaming an entry that is already
on EVA or assigned to a vehicle breaks that link. Guard on `!AssignedToVehicle && !Kia`.

That leaves the simple approach: let KSA build the roster as it likes, then rename the
entries whose `Character` is `Kerbal`. It covers the initial roster (`KittenRosterData.Create`,
20 entries) and every refill (`EnsureAvailableKitten`, 5 at a time) without patching either,
and the three fixed kittens — Hunter, Banjo and Polaris — are skipped for free, since their
character ids are `HunterKitten`, `BanjoKitten` and `PolarisKitten`.

`Name` and `NameHash` are plain public fields, so rewriting both is enough. `NameHash` is what
`KittenRosterData.Find` matches on, so the two must stay in step.

## Reference implementation

```csharp
using KSA;
using StarMap.API;

[StarMapMod]
public sealed class KerbalNamesMod
{
    private const string CharacterId = "Kerbal";
    private readonly Random _rng = new();
    private readonly List<string> _veteran = [], _specialM = [], _specialF = [], _commonM = [], _commonF = [];

    [StarMapAllModsLoaded]
    public void Load()
    {
        foreach (string line in File.ReadLines("kerbal_names.txt"))   // resolve against the mod folder
        {
            if (line.Length == 0 || line[0] == '#') continue;
            int sp = line.IndexOf(' ');
            if (sp < 0) continue;
            string name = line[(sp + 1)..];
            switch (line[..sp])
            {
                case "veteran":   _veteran.Add(name);  break;
                case "special-m": _specialM.Add(name); break;
                case "special-f": _specialF.Add(name); break;
                case "common-m":  _commonM.Add(name);  break;
                case "common-f":  _commonF.Add(name);  break;
            }
        }
    }

    [StarMapAfterOnFrame]
    public void AfterOnFrame(double _, double __)
    {
        List<KittenRosterEntryData> roster = Universe.KittenRoster.Kittens;

        bool any = false;
        foreach (KittenRosterEntryData e in roster)
            if (NeedsRename(e)) { any = true; break; }
        if (!any) return;                       // the normal case, every frame

        HashSet<KeyHash> used = [];
        foreach (KittenRosterEntryData e in roster) used.Add(e.NameHash);

        foreach (KittenRosterEntryData e in roster)
        {
            if (!NeedsRename(e)) continue;
            string name = NextName(used);
            used.Remove(e.NameHash);
            e.Name = name;
            e.NameHash = KeyHash.Make(name.AsSpan());
            used.Add(e.NameHash);
        }
    }

    // Already-renamed entries end in " Kerman", so this settles after one pass. Assigned and
    // KIA entries are left alone: EVADoor builds the EVA vehicle with the roster name as its id.
    private static bool NeedsRename(KittenRosterEntryData e) =>
        e.Character == CharacterId && !e.AssignedToVehicle && !e.Kia
        && !e.Name.EndsWith(" Kerman", StringComparison.Ordinal);

    private string NextName(HashSet<KeyHash> used)
    {
        foreach (string veteran in _veteran)    // Jebediah, Bill, Bob, Valentina first, once each
        {
            string full = veteran + " Kerman";
            if (!used.Contains(KeyHash.Make(full.AsSpan()))) return full;
        }

        for (int attempt = 0; attempt < 64; attempt++)
        {
            bool female = _rng.Next(2) == 0;
            bool special = _rng.Next(20) == 0;  // CrewGenerator.specialNameChance
            List<string> pool = (special, female) switch
            {
                (true, true)   => _specialF,
                (true, false)  => _specialM,
                (false, true)  => _commonF,
                (false, false) => _commonM,
            };
            string full = pool[_rng.Next(pool.Count)] + " Kerman";
            if (!used.Contains(KeyHash.Make(full.AsSpan()))) return full;
        }
        return "Kerbal " + Guid.NewGuid().ToString("N")[..6] + " Kerman";
    }
}
```

`mod.toml` then reads as both a content mod and a code mod, the way KSArmory's does:

```toml
name = "KSP Characters"
version = "0.1.0"

assets = [ "KerbalAssets.xml" ]

[StarMap]
EntryAssembly = "KerbalNames"
```

## Caveats

The rename lands a frame or so after the roster grows, so a name can be seen changing if the
crew list is open at that moment. Patching `KittenRosterData.CreateKitten` and `Create` with
Harmony postfixes avoids that and is otherwise the same logic. Entries already assigned keep
their kitten name for the life of that assignment, by design.

Surnames for the other ten languages KSP ships are not in this file; KSP's own `Kerman` comes
from `#autoLoc_6003018`, and only the `en-us` dictionary ships in a Steam install set to
English.
