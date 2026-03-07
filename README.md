# Skills Registry

A curated collection of **Agent Skills** for Dart and Flutter packages that do not ship their own skills. Use this repo together with the [skills](https://pub.dev/packages/skills) CLI so your AI coding assistant (Cursor, Claude Code, Cline, etc.) knows how to work with your stack.

## Why this exists

The [skills](https://pub.dev/packages/skills) CLI installs Agent Skills from your **pub dependencies**: when you run `skills get`, it finds `skills/` directories inside the packages you depend on and installs them into your IDE. Many popular packages (e.g. Riverpod, flutter-shadcn-ui) do not yet bundle skills. This registry provides skills for those packages so you can still get context-aware assistance.

## Contributing

Improvements and new skills (for other Dart/Flutter packages) are welcome. Make sure to follow the [Agent Skills](https://agentskills.io/specification) specification. Skills should be placed in a directory named after the Dart package, each skill should be prefaced with the package name. (See other skills as a reference.)
