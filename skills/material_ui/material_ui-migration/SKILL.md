---
name: material_ui-migration
description: Flutter 3.47+ standalone design packages (material_ui, cupertino_ui). Use BEFORE writing or fixing any Flutter UI code when pubspec.yaml lists material_ui or cupertino_ui or the SDK constraint is Flutter >= 3.47; when asked to migrate off package:flutter/material.dart or package:flutter/cupertino.dart, or about their deprecation; when an error says a type (ThemeData, ColorScheme, TextTheme, ...) from package:flutter/src/material can't be assigned to the same type from package:material_ui; or for localization delegates, MaterialUiCompatibilityBridge or migrate_design_widgets. Do not use to migrate projects below Flutter 3.47 unasked.
---

# Flutter standalone Material & Cupertino (material_ui / cupertino_ui)

Since **Flutter 3.47 (12 Aug 2026)** the Material and Cupertino libraries ship as standalone
pub.dev packages, versioned on their own (weekly releases):

| Old (in the SDK, frozen since 3.44) | New |
|---|---|
| `import 'package:flutter/material.dart';` | `import 'package:material_ui/material_ui.dart';` |
| `import 'package:flutter/cupertino.dart';` | `import 'package:cupertino_ui/cupertino_ui.dart';` |

The in-SDK libraries are frozen and scheduled for **deprecation in the November 2026 stable
release**. Class names and APIs are the same (1.0.0 matched the frozen code), but they are
**different libraries, so their types are not interchangeable**.

This postdates most models' training data. The packages are real; do not claim otherwise or
invent other names.

Source: https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui

## Step 1 — Decide which world the project is in

Read `pubspec.yaml` and grep the imports in `lib/`:

- **Already migrated** (`material_ui` / `cupertino_ui` in dependencies, files import them):
  every new or edited file uses the package imports. Never add `package:flutter/material.dart`
  or `package:flutter/cupertino.dart` — mixing them causes type errors (Step 4).
- **Not migrated, Flutter < 3.47** (SDK constraint below 3.47, no material_ui): keep using
  `package:flutter/material.dart`. Do not migrate or add the packages unless asked.
- **Not migrated, Flutter >= 3.47:** keep the existing style for small edits, and mention once
  that the migration is available. Migrate when asked (Step 2).

Widgets-only code (`package:flutter/widgets.dart`) is unaffected.

## Step 2 — Migrating an app

```console
flutter pub add material_ui
flutter pub add cupertino_ui        # if any file uses Cupertino
dart fix --apply --code=migrate_design_widgets
dart fix --apply                    # follow-up fixes, e.g. import sorting
```

- `migrate_design_widgets` rewrites the imports. If it doesn't add the dependencies, the
  `pub add` lines do. Add **cupertino_ui as a direct dependency** whenever a file imports it,
  even though material_ui depends on it (otherwise `depend_on_referenced_packages` fires).
- Show every changed file when the user asks for it, with the new imports.

### Localizations

Delegates moved into the packages. Replace the list and drop the
`flutter_localizations` import:

```dart
import 'package:material_ui/material_ui.dart';

MaterialApp(
  localizationsDelegates: GlobalMaterialLocalizations.delegates, // includes Cupertino + Widgets
  supportedLocales: const [Locale('en'), Locale('tr')],
)
```

`flutter_localizations` can come out of pubspec once nothing else imports it.

### Dependencies that still use the old library — compatibility bridge

If a third-party package still imports `package:flutter/material.dart`, wrap the app so it
still sees the theme and localizations:

```dart
MaterialApp(
  builder: (context, child) => MaterialUiCompatibilityBridge(child: child!),
  home: const HomePage(),
)
```

You can also wrap only the subtree that contains the legacy widget.

The bridge injects `Theme`/`MaterialLocalizations` data **only**. It **cannot** fix type
mismatches when the package's *API* takes or returns Material types (e.g. a parameter typed
`ColorScheme`, `TextTheme`, `FloatingActionButtonLocation`). Those packages must be upgraded
(or forked/replaced) first. Say this when you add the bridge.

### Package authors

Migrating a published package is a **major version bump** for its users.

## Step 3 — Writing new UI in a migrated project

```dart
import 'package:material_ui/material_ui.dart';

class SettingsPage extends StatelessWidget { ... }
```

```dart
import 'package:cupertino_ui/cupertino_ui.dart';

class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) => CupertinoPageScaffold(
        navigationBar: const CupertinoNavigationBar(middle: Text('Sign in')),
        child: ...,
      );
}
```

plus, in pubspec, if missing:

```yaml
dependencies:
  cupertino_ui: ^1.0.0
```

Everything else (widget names, `ThemeData`, `ColorScheme.fromSeed`, `showDialog`, …) is
written exactly as before.

## Step 4 — Diagnosing "X can't be assigned to X"

```
The argument type 'ThemeData/*1*/' can't be assigned to the parameter type 'ThemeData/*2*/'.
 - 'ThemeData/*1*/' is from 'package:flutter/src/material/theme_data.dart'
 - 'ThemeData/*2*/' is from 'package:material_ui/src/theme_data.dart'
```

1. The file that created the `/*1*/` value imports the legacy library. Find it (the error's
   call site → the function it calls → that file's imports, or grep for
   `package:flutter/material.dart` / `cupertino.dart`).
2. Change that import to `package:material_ui/material_ui.dart` (or cupertino_ui).
3. Run `dart fix --apply --code=migrate_design_widgets` to catch any other leftovers.
4. If the legacy type comes from a **dependency's API**, upgrade the dependency; the bridge
   won't help.

Never fix this with casts, `as`, import aliases or `hide` to keep both libraries. Don't
recommend removing material_ui and going back to the frozen, soon-deprecated SDK library.

## Checklist

- [ ] Imports match the project: material_ui/cupertino_ui if migrated; legacy only if < 3.47 and not migrated
- [ ] No file mixes `package:flutter/material.dart` with `package:material_ui`
- [ ] cupertino_ui is a direct dependency when imported
- [ ] Migration: `pub add` + `dart fix --apply --code=migrate_design_widgets`
- [ ] Localizations use `GlobalMaterialLocalizations.delegates`
- [ ] Legacy third-party dependency → `MaterialUiCompatibilityBridge`, with its type limitation stated

---

Maintained with eval results (with vs. without this skill) at https://github.com/lightM3/ai-skills.
