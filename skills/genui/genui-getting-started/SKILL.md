---
name: genui-getting-started
description: Flutter GenUI (package:genui, A2UI protocol) — AI-generated Flutter UI. Use BEFORE writing or fixing any code in a project whose pubspec.yaml lists genui, genui_a2ui, a2ui_core or json_schema_builder, or when asked to let an LLM/agent generate UI with Flutter widgets, set up a GenUI chat/conversation, define catalog items for the AI, or port code that uses older genui names (GenUiConversation, GenUiManager, A2uiMessageProcessor, GenUiSurface, CoreCatalogItems, ContentGenerator). genui's API changed in almost every 0.x release; this skill holds the 0.10 API. Not for plain chat UIs without genui.
---

# Flutter GenUI (genui 0.10.x, A2UI v0.9)

`package:genui` (publisher labs.flutter.dev, **0.10.4 as of Sep 2026**, still 0.x and breaking
often) lets an LLM build UI from a **catalog** of Flutter widgets using the **A2UI** protocol.
API docs: https://pub.dev/documentation/genui/latest/genui/ · changelog:
https://pub.dev/packages/genui/changelog

**Check `pubspec.yaml` first.** This file describes **0.10.x**. If the project pins an older
version, either upgrade it (see the rename table) or say the code targets an old API.

## Renamed / removed — never write these for 0.10

| Old | Now | Since |
|---|---|---|
| `GenUiManager` → `A2uiMessageProcessor` | `SurfaceController` | 0.6 / 0.7 |
| `GenUiConversation` | `Conversation` | 0.7 |
| `GenUiController` | `SurfaceController` | 0.7 |
| `GenUiSurface(host:, surfaceId:)` | `Surface(surfaceContext: controller.contextFor(id))` | 0.7 |
| `GenUiHost` / `GenUiContext` / `GenUiTransport` | `SurfaceHost` / `SurfaceContext` / `Transport` | 0.7 |
| `GenUiFunctionDeclaration` | `ClientFunction` | 0.7 |
| `AiClient` → `ContentGenerator` (subclass to call your LLM) | `A2uiTransportAdapter(onSend: ...)` | 0.4 / 0.10 |
| `CoreCatalogItems` / `CoreCatalog` | `BasicCatalogItems.asCatalog()` | 0.8 |
| `SimpleClientFunction.call` | `SynchronousClientFunction.executeSync` | 0.8 |
| `onSurfaceAdded:` / `onSurfaceDeleted:` callbacks | `conversation.events` stream | 0.7+ |
| `UserMessage.text(t)` | `ChatMessage.user(t)` | 0.7+ |
| `SurfaceController.store`, `DataModelStore`, `SurfaceRegistry.updateSurface` | `SurfaceController.handleMessage` | 0.10 |

## Core objects

```dart
SurfaceController({required Iterable<Catalog> catalogs, Duration pendingUpdateTimeout})
A2uiTransportAdapter({Future<void> Function(ChatMessage message)? onSend})
  ..addChunk(String text)        // feed LLM text chunks (parsed into A2UI messages)
  ..addMessage(A2uiMessage m)    // feed an already-parsed message
Conversation({required SurfaceController controller, required Transport transport})
  ..events   // Stream<ConversationEvent>
  ..state    // ValueListenable<ConversationState>
  ..sendRequest(ChatMessage message)
Surface({required SurfaceContext surfaceContext, WidgetBuilder? defaultBuilder, ActionDelegate actionDelegate})
ChatMessage.user(text) / ChatMessage.model(text) / ChatMessage.system(text); message.text
```

Events (sealed under `ConversationEvent`): `ConversationSurfaceAdded(surfaceId, definition)`,
`ConversationSurfaceRemoved`, `ConversationComponentsUpdated`, `ConversationContentReceived`,
`ConversationWaiting`, `ConversationError`.

The flow: user text → `conversation.sendRequest` → your `onSend` calls the LLM → you pipe the
streamed text into `adapter.addChunk` → genui parses A2UI → `SurfaceController` creates/updates
surfaces → `ConversationSurfaceAdded` → you render `Surface`. User actions in a surface go back
through `controller.onSubmit` to the conversation automatically.

## Chat screen (0.10)

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:genui/genui.dart';

class AssistantPage extends StatefulWidget {
  const AssistantPage({super.key});
  @override
  State<AssistantPage> createState() => _AssistantPageState();
}

class _AssistantPageState extends State<AssistantPage> {
  final _llm = LlmClient();
  final _catalog = BasicCatalogItems.asCatalog();
  late final _controller = SurfaceController(catalogs: [_catalog]);
  late final _adapter = A2uiTransportAdapter(onSend: _send);
  late final _conversation = Conversation(controller: _controller, transport: _adapter);
  late final String _systemPrompt =
      PromptBuilder.chat(catalog: _catalog).systemPromptJoined();
  final _history = <({String role, String text})>[];
  final _surfaceIds = <String>[];
  StreamSubscription<ConversationEvent>? _events;
  bool _busy = false;

  @override
  void initState() {
    super.initState();
    _events = _conversation.events.listen((event) {
      switch (event) {
        case ConversationSurfaceAdded():
          setState(() => _surfaceIds.add(event.surfaceId));
        case ConversationSurfaceRemoved():
          setState(() => _surfaceIds.remove(event.surfaceId));
        case ConversationError():
          // show a snackbar / log
          break;
        default:
          break;
      }
    });
  }

  Future<void> _send(ChatMessage message) async {
    _history.add((role: 'user', text: message.text));
    final reply = StringBuffer();
    await for (final chunk in _llm.streamText(systemPrompt: _systemPrompt, history: _history)) {
      reply.write(chunk);
      _adapter.addChunk(chunk);
    }
    _history.add((role: 'model', text: reply.toString()));
  }

  Future<void> _submit(String text) async {
    setState(() => _busy = true);
    try {
      await _conversation.sendRequest(ChatMessage.user(text));
    } finally {
      if (mounted) setState(() => _busy = false);
    }
  }

  @override
  void dispose() {
    _events?.cancel();
    _conversation.dispose();
    _adapter.dispose();
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        body: Column(children: [
          Expanded(
            child: ListView(children: [
              for (final id in _surfaceIds)
                Surface(key: ValueKey(id), surfaceContext: _controller.contextFor(id)),
            ]),
          ),
          if (_busy) const LinearProgressIndicator(),
          TextField(onSubmitted: _submit),
        ]),
      );
}
```

- **System prompt:** the LLM must be told how to emit A2UI for this catalog —
  `PromptBuilder.chat(catalog: ...)` (or `PromptBuilder.custom(catalog:, allowedOperations:)`)
  `.systemPromptJoined()`, plus `catalog.systemPromptFragments`. The 0.10.0 changelog made
  `PromptBuilder.chat/custom` async; the 0.10.4 API docs list them returning `PromptBuilder`.
  If the analyzer says it returns a `Future`, `await` it (e.g. build the prompt in `initState`
  before the first send).
- `A2uiTransportAdapter.flush()` exists for draining buffered text; call it after the stream
  ends if surfaces appear only on the next message.
- iOS/macOS: add the `com.apple.security.network.client` entitlement for network LLM calls on
  macOS; any Firebase/Gemini client works as long as you pipe its text into `addChunk`.

## Custom catalog items

```dart
import 'package:genui/genui.dart';
import 'package:json_schema_builder/json_schema_builder.dart';

final productCard = CatalogItem(
  name: 'ProductCard',
  dataSchema: S.object(
    description: 'A product with price.',
    properties: {
      'name': S.string(description: 'Product name'),
      'price': S.number(description: 'Price as a number'),
      'currency': S.string(description: 'ISO 4217 code, e.g. EUR'),
      'imageUrl': S.string(description: 'Optional image URL'),
      'onSale': S.boolean(description: 'Whether the product is discounted'),
    },
    required: ['name', 'price', 'currency'],
  ),
  widgetBuilder: (itemContext) {
    final data = itemContext.data as Map<String, Object?>;
    return Card(child: ListTile(
      title: Text(data['name'] as String),
      subtitle: Text('${data['price']} ${data['currency']}'),
    ));
  },
);

final Catalog assistantCatalog =
    BasicCatalogItems.asCatalog().copyWith(newItems: [productCard]);
```

- `widgetBuilder` takes **one** `CatalogItemContext` (`data`, `id`, `type`, `surfaceId`,
  `buildContext`, `buildChild(id)`, `dispatchEvent(...)`, `dataContext`, `getComponent`,
  `reportError`). The old named-parameter builder (`({required data, required id, ...})`) is gone.
- Catalogs are immutable: `copyWith(newItems:)` (same name replaces), `copyWithout(itemsToRemove:)`,
  or `Catalog([...items], functions: [...], catalogId: ..., systemPromptFragments: [...])`.
- Give every schema property a `description` — the LLM reads them. Add `exampleData` for
  better generations. Container children: render with `itemContext.buildChild(childId)`.
- For buttons/inputs prefer the basic items (`BasicCatalogItems.button`, `.textField`, …)
  over re-implementing event wiring.

## Rules

- Never use names from the rename table; never invent APIs (no `GenUiClient`,
  `surfaceStream`, `ContentGenerator` subclasses). If unsure of a signature, say so and point
  to the API docs above.
- Keep LLM transport in `onSend`; don't parse A2UI JSON by hand or build widgets from raw JSON.
- Dispose conversation, adapter, controller and the events subscription.

---

Maintained with eval results (with vs. without this skill) at https://github.com/lightM3/ai-skills.
