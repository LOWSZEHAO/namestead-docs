---
layout: default
---

Keep a project clean, consistent and manageable as it grows.

Large projects drift. Names stop matching whatever convention they started with, assets end up
in the wrong folders, and nothing notices until someone goes looking. Namestead defines what
correct looks like as a chain of rules, then uses that one definition to rename in bulk, to
validate what already exists, and to report on a whole project. Nothing is renamed until it has
been previewed.

Unreal already ships a batch renamer. This is not one of those.

| | |
| --- | --- |
| **Rename** | Composable rules over a selection or a folder, with a live preview and collision checking |
| **Validate** | The same rules as a project-wide convention, enforced through the engine's own Data Validation |
| **Audit** | Whole-project reports: names, unreferenced assets, empty folders, duplicate imports |
| **Organize** | A folder plan that says where each class of asset belongs, previewed and applied like a rename |
| **Automate** | All of it headless, with CSV reports and exit codes a build step can act on |

## Contents
{:.no_toc}

<div class="toc" markdown="1">
* TOC
{:toc}
</div>

## Getting started

Bought it on Fab? Open the Epic Games Launcher, go to **Unreal Engine > Library > Fab Library**,
find Namestead and press **Install to Engine**, then choose the engine version. Epic compiles the
plugin, so there is nothing for you to build: no compiler, no Visual Studio, and the project does
not have to be a C++ one. Start that engine and tick Namestead on under **Edit > Plugins**.

The Fab window inside the editor sells plugins but will not install them — the launcher is the
only route. [Installation](#installation) has the rest, including building from source.

Then, in about two minutes:

1. **Open it.** **Tools > Namestead** loads what is selected in the Content Browser, and so
   does right-clicking a selection and picking **Open in Namestead**. Select nothing and it
   takes the folder you are in instead, which is the only time a folder is consulted: an
   explicit selection is the batch, and never has anything added to it.
2. **Load a starting point.** **Load Preset > Unreal Standard** gives each asset the prefix for
   its class — `SM_`, `BP_`, `MI_` and so on, eighteen classes in all. Anything else is left
   alone rather than guessed at, so a batch of ordinary assets comes back Unchanged. Maps are a
   special case: a World Partition level never enters the batch at all, for the reason under
   [Renaming behavior](#renaming-behavior).
3. **Read the preview.** Every row shows the current name, the proposed name, and a status. The
   line under the table says how many assets a rename would *also rewrite* — renaming a
   referenced asset repoints the things that point at it, and those were never selected.
4. **Rename.** The button says how many will actually change, which is not how many are
   selected. Anything validation refused is excluded from that number and says why in its row.
5. **Make it the project's rule.** **Load Preset > Set These Rules as the Convention** saves the
   chain as an asset and points the project at it. Those rules then run wherever the engine
   validates — the Validate Assets action, and the DataValidation commandlet on a build machine.
   All but any rule that depends on an asset's position in a batch, which cannot judge one asset
   alone and is left out.

No asset is renamed until you press Rename, and building the preview only reads. Three other
things here do write, all deliberately: Save Preset and Set These Rules as the Convention create
an asset (and the convention also rewrites the project's `DefaultEditor.ini`), and Export Report
writes a CSV.

Once a convention is set, **Project Health** reports the whole project against it, and the
button that then reads **Review N Naming Fix(es)** turns what it found back into an ordinary
preview.

## Rules

Rules are composable and order dependent. Each one receives the name produced by the rule
before it:

```
Door                    original
SM_Door                 asset type prefix
ENV_SM_Door             prefix "ENV_"
ENV_SM_Door_001         sequential numbering
```

| Rule | Options |
| --- | --- |
| Prefix / Suffix | Text, case sensitivity, skip if already present |
| Find and Replace | Case sensitivity, whole-name match, first occurrence only |
| Regex Replace | Capture groups as `$1` to `$9`, first match only, case sensitivity |
| Sequential Numbering | Numbers or letters (`a`, `b`, `c`), start, increment, padding, separator, prepend or append |
| Asset Type Prefix | `SM_`, `SK_`, `M_`, `MI_`, `T_`, `BP_` and so on, overridable per class |
| Change Case | lower, UPPER, Title, Pascal, camel, snake, kebab |
| Remove Prefix / Suffix | By exact text, up to a separator, or a character count |
| Remove Numbering | Leading or trailing digits, with or without their separator |
| Clean Up Separators | Spaces and mixed separators to one character, collapse runs, trim ends |
| Name Pattern | `{NAME}`, `{ORIGINAL}`, `{TYPE}`, `{INDEX}`, `{FOLDER}` |

A class with no mapping in the asset type rule is left untouched rather than guessed at, so a
mixed selection never produces a name that was not asked for.

Rule chains save as preset assets in the project, so a team versions its naming alongside the
assets it describes.

Three chains are built in, so Load Preset is never empty on a first run. It lists them under
**Built In**, above whatever the project has saved:

| Preset | Does |
| --- | --- |
| **Unreal Standard** | Gives every asset the prefix for its class |
| **Unreal Standard, PascalCase** | The type prefix, then PascalCase for the rest. `SM_stone_wall` to `SM_StoneWall` |
| **Tidy Imported Names** | Unifies mixed separators, PascalCases, then adds the prefix. `wall--final__01` to `SM_WallFinal01` |

Each is built in code rather than shipped as an asset. An asset carries the package version of
whatever engine saved it, and an engine refuses anything stamped newer than itself, so a preset
saved in 5.8 would be invisible on 5.4 through 5.7. Picking one fills the panel with a fresh
copy of its rules, which you can then edit and save as a preset of your own.

## Level actors

The same rules rename actors placed in a level. Right-click a selection in the viewport or the
Outliner and pick **Batch Rename Actors**, or switch the panel to **Level Actors**.

A rule only ever sees strings, so the whole chain works unchanged. What differs is either
side of it, and the panel says so rather than pretending otherwise: a label may contain
spaces, two actors may share one, and nothing anywhere references it, so there is no collision
to refuse and no collateral to warn about.

**Renaming actors can be undone.** `SetActorLabel` calls `Modify`, so a batch runs inside one
transaction and steps back with a single `Ctrl+Z`. A batch asset rename cannot: Namestead
offers no `Ctrl+Z` for one because the engine's rename path opens no transaction and writes to
disk before it returns. That is why one is recorded to a CSV and the other does not need to be.

Wrapping `IAssetTools::RenameAssets` in an `FScopedTransaction` does not help, and Epic's own
renamer wraps its rename in one anyway. The reason is structural rather than an omission: undo
replays serialized property data, and an object's name and outer are not property data, so even
where the engine does record the object there is nothing in the record that puts the name back.

## Naming conventions

A convention is a rule set. An asset complies when running the rules over its name changes
nothing, and the violation message is the name the rules would have produced. The thing that
detects a problem is therefore the same thing that fixes it, with no second pattern language
to keep in step.

Point Project Settings > Plugins > Namestead at a rule set and validation runs wherever the
engine already validates: the Validate Assets action, validation on submit, and the
DataValidation commandlet on a build machine.

Every rule can be scoped to the classes it applies to, which is how one chain gives each kind
of asset its own name shape:

| Rule | Applies to | Setting |
| --- | --- | --- |
| Prefix | StaticMesh | `SM_` |
| Prefix | Material | `M_` |
| Prefix | Texture2D | `T_` |

A rule listing no classes applies to everything, and an asset no rule claims is left alone.
Matching is on the exact short class name, so a rule scoped to `Material` does not cover
`MaterialInstanceConstant` — list both when you mean both.

**A convention has to be idempotent, and this is the trap.** An asset complies when running the
rules over its name changes nothing, so a rule that keeps changing an already-correct name
produces a convention no asset can ever satisfy. Prefix is safe: `bSkipIfAlreadyPresent` is on
by default, so `SM_Door` comes back unchanged. **Name Pattern is not** — it has no such guard,
so a pattern of `SM_{NAME}` turns `SM_Door` into `SM_SM_Door`, and the asset is reported as a
violation forever. Use Name Pattern for renaming, and prefix rules for conventions.

Rules that depend on an asset's position in a batch, such as Sequential Numbering, cannot be
checked one asset at a time and are left out of validation. The health report says when that has
happened — the Naming row carries the note, in the panel and in the commandlet — rather than
reporting a partial check as a clean one.

## The panel

Four things the panel points at, chosen with the radio buttons at the top left: **Assets**,
**Level Actors**, **Project Health**, and **Organize**. The first two edit a rule chain; the
last two report on the project and act on what they find.

Project Health runs every whole-project report and shows the summary above the findings.
Nothing there writes. When it finds naming violations, **Review N Naming Fix(es)** loads them as an
ordinary batch in the Assets target, where they are previewed and executed by the same code
every other rename goes through — so there is no second execution path, and nothing is written
until you have seen it.

Load Preset also shows what the project currently validates against, and **Set These Rules as
the Convention** saves the chain in the panel as an asset and points the project at it. That is
the whole loop: edit rules, adopt them, and validation runs wherever the engine already
validates.

Open **Tools > Namestead**, or run `Namestead.Open` from the console, or right-click a
selection in the Content Browser and pick **Open in Namestead**. The panel picks up the
Content Browser selection, or everything inside a selected folder, and shows what each asset
would become before anything is written. The button reports how many assets will actually be
renamed, not how many are selected.

## Automating

Everything the panel does to assets, a build machine can do: renaming, organizing, auditing and
project health. Renaming level actors is the one exception, because a label lives in a level
rather than in the asset registry. Renaming runs headless, reporting what it
would do and changing nothing unless `-execute` is passed:

```
UnrealEditor-Cmd.exe <project> -run=NamesteadRename -path=/Game/Environment -typeprefix
```

```
CURRENT                                  NEW                                      STATUS
SM_wall_001                              M_SM_wall_001                            Valid
wall_003                                 M_wall_003                               Valid
door                                     BP_door                                  Already exists (An asset already exists at ...)

15 asset(s), 13 renameable.
6 other asset(s) reference these and will be resaved.
Dry run. Pass -execute to apply.
```

| Argument | Meaning |
| --- | --- |
| `-path=` | Where to look. Join several with `+` |
| `-class=` | Restrict to an asset class, such as `Material`. Join several with `+`. A name that cannot be resolved fails the run |
| `-recursive=` | Include subfolders. On by default; `-recursive=false` looks in the given folders only |
| `-prefix=` `-suffix=` | Add text |
| `-find=` `-replace=` | Substring replacement |
| `-typeprefix` | Prefix by asset class |
| `-number` | Append a sequential counter |
| `-preset=` | Drive the run from a saved rule set, such as `/Game/Conventions/DA_Naming.DA_Naming` |
| `-convention` | Drive the run from the project's naming convention |
| `-organize` | Move assets into the folders the project's folder plan gives them, instead of renaming. Cannot be combined with rules |
| `-sort=` | `name`, `type` or `path`, which decides the numbering order |
| `-csv=` | Write the report. On a dry run this is the preview, so it can be reviewed or approved before anything is applied |
| `-execute` | Apply the rename |

Both exports share a column layout, so a dry run and the run that followed it can be diffed
against each other. A report that was asked for and could not be written exits non-zero rather
than passing quietly.

`-convention` is what closes the loop. The audit reports what breaks the project's convention;
the rename fixes it from that same rule set. Detection and repair are one definition, so a
build step can do both:

```
UnrealEditor-Cmd.exe <project> -run=NamesteadAudit -naming -strict
UnrealEditor-Cmd.exe <project> -run=NamesteadRename -path=/Game -convention -execute
```

## Project health

One pass over a project, reported as a summary rather than five lists:

```
UnrealEditor-Cmd.exe <project> -run=NamesteadAudit -health
```

```
PROJECT HEALTH
785 asset(s) scanned.

  Naming                         1 of 785 (99% clean)
  Organization                   15 of 15 (0% clean)
  Potentially unreferenced       778 found
       Potentially unreferenced. An asset reached only from C++, from a path built at
       runtime or from config has no recorded referencer. Review rather than delete.
  Duplicate import candidates    nothing found
       Candidates. Identical bytes at import time does not mean interchangeable now, and
       only one of a group may be referenced.
  Empty folders                  nothing found

794 finding(s).
```

Three things about that output are deliberate.

**A percentage appears only where there is something honest to divide by.** Organization's
denominator is the assets the folder plan actually has an opinion about, not the whole project,
or a plan covering one class would score a project 99% organised on the strength of saying
almost nothing. The other three report a count and no percentage — duplicates counts *groups*
rather than assets, and an unreferenced asset is not necessarily a defect, so a score there
would be read as a defect rate it is not. Naming divides by every asset scanned, which is worth
knowing if your rules are class-scoped: an asset no rule claims can never be a violation, so it
counts towards the total and always towards the clean side of it.

**Not configured is never reported as clean.** A project with no naming convention set has not
passed the naming check, it has declined to take it, and those must not look the same. If
neither a convention nor a folder plan is set, `-health` says so and exits non-zero — the other
three categories need no configuring and would otherwise report a comfortable number about a
project that has never declared a standard at all.

**A partial check says so.** A convention containing a rule that depends on an asset's position
in a batch cannot be fully checked one asset at a time, so that part is left out and the row is
marked rather than passing quietly.

## Auditing

Four of the five whole-project reports, all read-only. The fifth, `-organization`, is under
[Organizing](#organizing):

```
UnrealEditor-Cmd.exe <project> -run=NamesteadAudit -naming -unused -empty -duplicates
```

Names that break the convention, assets nothing references, content folders holding nothing,
and candidates for the same source file having been imported more than once. With `-strict` it
exits non-zero when anything is found, which is what makes it usable as a build step.

Each report states its own limits, and two of them are worth stating here, because both are
easy to read as more certain than they are.

**Potentially unreferenced** is not *unused*. An asset reached only from C++, from a path built
at runtime, or from a config entry has no recorded referencer and appears in this list. It is
for review, not for deletion. Levels are the exception: nothing points at a map, so maps are
treated as roots of the reference graph and are never listed.

**Duplicate import candidates** are not confirmed duplicates. Grouping is by the MD5 the engine
recorded at import, which proves two assets came from identical bytes on disk at that time. It
does not prove they are interchangeable now — either could have been edited since, and only one
may be referenced. Nothing here offers to delete anything, which is deliberate.

`-csv=` covers the naming report, the only one of the four that is a table rather than a list.
A run that produces no naming report — the other passes on their own, or no convention set —
says so and exits non-zero, rather than leaving a file that reads like a clean result.

## Organizing

A move in Unreal is a rename with a different package path, so organizing produces ordinary
rename plans and shares the same validation, preview, execution and reporting. A folder plan
asset says where each class belongs; anything it says nothing about stays where it is, so a
partial plan covering only the classes you care about is a reasonable thing to have.

Create one from the Content Browser's **Add** menu, under **Miscellaneous**, as **Namestead
Folder Plan**. It starts populated with a conventional layout rather than empty, so the shape is
obvious and editing is the only step. Point Project Settings > Plugins > Namestead at it, and:

```
UnrealEditor-Cmd.exe <project> -run=NamesteadAudit -organization
```

```
ORGANIZATION: 15 asset(s) are not in the folder the plan gives them.
  /Game/NamesteadTest/BP_Door_001 -> /Game/Blueprints
  /Game/NamesteadTest/M_wall_003_005 -> /Game/Art/Materials
```

Assets already sitting where the plan wants them are not listed, and a move validation refused
is listed with the reason rather than quietly dropped.

To apply them, switch the panel to **Organize**. It shows what would move, where to, and what
each move would drag along with it, and the button says **Move N**. With a selection loaded it
checks that; with nothing loaded it checks the whole project. From Project Health, the
**Review N Move(s)** button goes straight there.

Headless, moving is the rename commandlet with a different source of plans:

```
UnrealEditor-Cmd.exe <project> -run=NamesteadRename -path=/Game -organize -execute
```

A move *is* a rename with a different package path, so it goes through the same validation,
collision checking, referencing count, run report and history CSV as any
other rename. Which also means the same limit: **a move cannot be undone with Ctrl+Z**, and the
CSV is the record of what happened.

`-organize` refuses to run alongside rules rather than quietly ignoring half of what was asked
for. Moving and renaming in one pass is two operations; run them as two steps.

## Architecture

```
Selection -> Rule chain -> Proposed names -> Validation -> Preview -> Execution
             (pure)                          (registry)             (IAssetTools)
```

Name generation is separate from execution, so previewing cannot modify an asset. Renaming
goes through `IAssetTools::RenameAssets` rather than touching `.uasset` files, which leaves
redirectors, reference fix-up and asset registry state to the engine.

Every rule implements a single method:

```cpp
virtual FString Apply(const FNamesteadRuleContext& Context) const;
```

The context carries the original name, the name so far, the asset's class, its folder and its
index in the batch. A rule has no access to the asset registry, the filesystem or a `UObject`,
so the entire rule layer can be exercised without creating an asset. Rules derive from
`UNamesteadRenameRule` and are marked `EditInlineNew`, so their parameters are editable
through a details view without a bespoke panel per rule.

Adding a rule means subclassing that base and overriding `Apply`. Nothing in the UI changes.

### Rules from your own module

A rule does not have to live in this plugin. `UNamesteadRenameRule` is exported and lives in
`Public/`, so a studio can keep its own rules in its own editor module:

```cpp
// MyStudioRule.h
#pragma once

#include "CoreMinimal.h"
#include "Rules/NamesteadRenameRule.h"

#include "MyStudioRule.generated.h"

UCLASS(DisplayName = "Studio Department Prefix")
class UMyStudioRule : public UNamesteadRenameRule
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Rule")
    FString Department;

    virtual FString Apply(const FNamesteadRuleContext& Context) const override
    {
        return Department + TEXT("_") + Context.CurrentName;
    }
};
```

The generated header must be the last include, and must match the file name. Add
`"NamesteadEditor"` to your module's `PrivateDependencyModuleNames`, and `Namestead` to your
`.uplugin` dependencies. **There is no registration call.** The add-rule dropdown is the
engine's own class picker filtered by the base class, so a rule appears as soon as its module
is loaded, and the pipeline calls a virtual without ever asking what type a rule is.

Two things worth knowing. Override `IsPositionDependent` and return true if `Apply` reads
`Context.Index`, or a convention containing your rule will report every asset in the project as
a violation. And a preset referencing a rule whose module is absent loads with that rule
missing — Namestead reports how many went missing rather than quietly enforcing a shorter
chain.

Rules cannot be written in Blueprint. `Apply` is a plain virtual, deliberately: the rule layer
has no access to the asset registry, the filesystem or a `UObject`, and that is what lets all
of it be tested without creating an asset.

## Renaming behavior

Two engine behaviors shape the design.

A batch **asset** rename cannot be undone with `Ctrl+Z`. Nothing in the engine's rename path
opens a transaction, and the operation writes to disk before it returns. Namestead therefore
records every run to the project's `Saved` folder as a CSV, automatically, whether it came
from the panel or the commandlet.

That record is an audit trail, and it is also the way back. **History**, then a run, then **Load
This Run in Reverse**: the report is read, every rename it made is pointed the other way, and the
result arrives as an ordinary preview. The same validation, the same collision check, the same
referencing count, the same button. Nothing is renamed until you press it.

It is still not an undo, and the difference is worth understanding rather than glossing. It
resolves every asset against the project as it stands now, not as the report remembers it, so
anything renamed again, moved or deleted since that run is simply not there and is left out —
the panel says how many. A name that something else has taken in the meantime is refused in its
own row, like any other collision. What comes back is what can come back, and you see exactly
that before anything runs.

It works after the editor has been closed and reopened, which is the part Ctrl+Z cannot do.

Reversals are walked back to front. Within one batch a later rename may have freed the name an
earlier one wanted, so undoing in the order it happened would walk into that collision instead of
out of it.

Renaming actors is the exception, and the reason the two are separate: a label change goes
through `Modify`, so it undoes properly and needs no record.

Renaming an asset referenced only by other *assets* usually leaves no redirector behind. The
engine loads those packages, repoints them and resaves them in place.

An asset referenced by a **level** is different, and it is the ordinary case in a game project.
The engine deliberately refuses to load map packages for reference fix-up and leaves a
redirector instead, unconditionally — so renaming a mesh or material that is placed in a level
does leave one. So do assets in a Content Browser collection, and referencers that cannot be
resaved, such as files checked out by another user.

The consequence is that renaming one asset writes to others that were never selected. Renaming
six materials in the test project rewrote six material instances that referenced them.
Namestead counts those before acting and reports the number, because it is the part of a bulk
rename that is easiest to be surprised by.

### What is never renamed

Some things are dropped before the batch is even built, so they leave no row in the table. The
log lists how many, at Verbose.

**World Partition and One File Per Actor maps.** A level of this kind keeps its actors in
generated packages under a folder named after the map, and nothing in the engine moves that
folder when the map is renamed. `AssetRenameManager` deliberately brings a level's build data
along and says so in its own comment, but has no equivalent for external actors — so the
renamed map looks for a folder still sitting under the old name and opens with none of its
actors in it. The rename reports success, and there is no undo. Every UE5 template map is World
Partition, so this is the first thing a stock project runs into. Unreal's own
`WorldPartitionRenameDuplicateBuilder` is the tool that moves a world *with* its external
packages; use that instead.

**The generated packages themselves**, under `__ExternalActors__` and `__ExternalObjects__`.
Their names are derived from the actor they belong to, so renaming one breaks the link.

**Engine, script and temp content** — `/Engine`, `/Script` and `/Temp` are not the project's to
modify. Plugin content is not restricted; it is yours.

**Redirectors.** A redirector is a stub left by an earlier rename, so renaming one moves the
signpost rather than the asset. A name held only by a redirector counts as free, though, so
renaming *onto* it is allowed.

Two more are refused later, in validation, and those do get a row explaining themselves: a
change that differs only in letter case, and a name another asset in the same batch is itself
moving out of.

## When something looks wrong

**The panel is empty when I open it.** It reads the Content Browser selection at the moment it
opens, and deliberately does not count the folder you happen to be browsing — that would turn
opening the tab into a request for the whole of `/Game`. Select something and press **Use
Content Browser Selection**.

Or everything you picked is something Namestead does not rename — a redirector, engine content,
a generated World Partition package, or a map whose actors live outside it. Those are dropped
before the table is built, so they leave no row at all. See
[What is never renamed](#what-is-never-renamed).

**The button says Rename 0.** Either nothing is loaded and no rule has been added — the line
above the button says which — or every proposed name is identical to the current one or was
refused. In that case the Status column says which for each row, and the reason is in its
tooltip.

**It rewrote assets I did not select.** It renamed only what you selected. Renaming a referenced
asset makes the engine load the packages that point at it, repoint them and resave them in
place — that is the engine's behaviour, not this plugin's, and it is why the line under the
table counts them before you press anything.

**Ctrl+Z did not undo an asset rename.** It cannot. The engine opens no transaction on that
path and writes to disk before returning. Every run is recorded to a CSV under the project's
`Saved` folder, which is the record of what happened, not an undo. Renaming *actors* does undo,
because a label change goes through `Modify`.

**A row says "Case only change".** Namestead refuses to rename `wall` to `Wall` in one step,
on every platform. The engine renames the package in memory, then cannot write the new file
beside the old one on a case-insensitive filesystem, and the asset is left on disk under neither
name — that was measured against a real asset, which did not come back. Rename to a third name,
then to the one you want.

**A preset has fewer rules than when it was saved.** One of its rules comes from a module that
is not present on this machine, so it deserialised as an empty entry. Namestead logs how many
went missing rather than quietly enforcing a shorter chain.

**Project Health says "not configured".** That category has nothing to check against — no
naming convention, or no folder plan. It is reported separately from "clean" on purpose: a
project that has opted into no checks has not passed them.

**"Set These Rules as the Convention" is greyed out.** Either the rule list is empty, since an
empty convention would pass every asset and look exactly like a project that complies, or you
are in Project Health or Organize, where a report is sitting where the rules were and you cannot
see what you would be adopting. The line above the entry says which.

**Naming reports almost every asset in the project.** Something in the convention proposes a
name almost nothing already has. Two usual causes. A custom rule that reads `Context.Index`
without overriding `IsPositionDependent` is evaluated at index 0 for every asset and proposes
the same name for all of them. Or a rule that *is* correctly position-dependent is dropped from
validation, and what the remaining rules then produce no longer matches the names your assets
carry — a counter placed *before* a prefix rule does this. Check what the New Name column
actually says; it is the name the convention wants, and it usually makes the cause obvious.

When a convention contains position-dependent rules at all, the Naming row is marked **partly
checked** and names what was skipped, rather than reporting a clean project without
qualification.

## Testing

Fifty six test groups cover the rules, discovery, validation, the preview the panel shows,
execution reporting, presets, conventions, auditing, organizing and actor renaming. They create no assets, so
they run anywhere:

```
UnrealEditor-Cmd.exe <project> -ExecCmds="Automation RunTests Namestead; Quit" -unattended -nullrhi
```

Or from Tools > Session Frontend > Automation, under `Namestead`.

## Requirements

Installed from Fab:

- Unreal Engine 5.6, 5.7 or 5.8
- Windows, on a Win64 editor

That is the whole list. Epic builds the plugin against each engine version and ships the
binaries, so there is no compiler to install and the project does not have to be a C++ one.

Building from source instead — needed only on a source-built or custom engine, or if you want to
modify the plugin:

- A C++ project, because a plugin with no matching binaries is compiled by UBT alongside it
- Visual Studio 2022 with the Game development with C++ workload

The module declares `"PlatformAllowList": [ "Win64" ]`. There is no platform-specific code in
it — everything goes through `FPaths` and `IFileManager` — so it would very likely build and
run on a macOS or Linux editor. It is restricted because it has never been run on one. That is a
statement about what has been tested rather than about what works.

This is about the editor you work in, not about what your game ships to. Namestead is an editor
plugin; it is not part of a packaged build and does not touch your target platforms. A project
shipping to Mac, PlayStation or Switch uses it exactly the same way.

## Installation

**From Fab.** Epic Games Launcher > Unreal Engine > Library > **Fab Library** > Namestead >
**Install to Engine**, then choose the engine version. Only versions Namestead supports appear,
and only ones you already have. Run it once per engine version you work in.

It installs into the engine rather than into your project, so it is then available to every
project on that engine version — but switched off until you say otherwise. Start the editor,
open **Edit > Plugins**, find Namestead, tick it, and restart when asked.

Two things that catch people out:

- There is no zip to download from fab.com. Unreal-format products come through the launcher.
- The Fab window *inside* the editor will sell you a plugin but will not install one. Its `+`
  button adds content packs to the open project; on a plugin it does nothing. Close the editor
  and use the launcher.

**From source.** Copy the plugin folder into the project's `Plugins` directory and rebuild. The
module compiles with the project and the panel appears under Tools. This is the path for a
source-built engine, where Epic's binaries will not match yours, and for modifying the plugin.

```
<YourProject>/Plugins/Namestead/
```

Moving a Fab copy into a project to change it? Delete its `Binaries` and `Intermediate` folders
first, or the prebuilt binaries load instead of your edits.

## Compatibility

| Engine | On Fab | Tests |
| --- | --- | --- |
| 5.8 | Yes — developed against | 56 of 56 pass |
| 5.7 | Yes | 56 of 56 pass |
| 5.6 | Yes | 56 of 56 pass |
| 5.5 | No | 56 of 56 pass |
| 5.4 | No | 56 of 56 pass |

Epic builds a submitted plugin against the three latest engine versions by default, which is why
the store offers 5.6 and up.

5.4 and 5.5 are not for sale and are not abandoned either. The suite runs on them on every pass,
and the plugin builds and works there from source. If you are on one of them and want it, ask —
older versions can be added to a listing by request.

The source contains no engine version guards. Two scripts rebuild this table, and they check
deliberately opposite things:

`Scripts/BuildAgainstEngines.ps1` packages against every installed engine with the same
`RunUAT BuildPlugin` a marketplace submission would use, plus `-StrictIncludes`, which turns
off the precompiled header and the unity build so every header has to compile on its own.

`Scripts/RunTestsAgainstEngines.ps1` builds a throwaway host project per engine and runs the
whole suite in each, with `-DisableAdaptiveUnity`, which forces translation units *together*.

Neither finds what the other does. Unity concatenates files, so it catches two file-local
helpers colliding under one name and hides a header that leans on its neighbour; strict
includes takes them apart, so it catches the second and cannot see the first. Both have caught
a real defect that shipped past every other check.

The second exists because compiling is the weaker claim. It found a real defect the first could
not: three rule sets that shipped as assets loaded on 5.8 and on nothing older, because a
package carries the version of the engine that saved it. They are built in code now.

## Support

Something broken, or something missing? Email **lowszehao.dev@gmail.com**. Include the engine
version, and if a rename went wrong, the CSV from Export Report — it records what was attempted
and what the engine said about each row, which is usually enough to find the cause without a
back-and-forth.

## Licence

Namestead is commercial software. Copies bought on Fab are covered by Epic's Content Licence
Agreement, and that is what governs your use of it. You receive the full source, and it is there
to be read and adapted to your project.

Copyright (c) 2026 Low Sze Hao. All rights reserved.
