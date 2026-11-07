---
name: flutter_bloc-best-practices
description: Flutter Bloc/Cubit guide. Use BEFORE writing any code in a Flutter project that depends on flutter_bloc or bloc_test — a new feature, page, state, Cubit, Bloc, stream or search handling, or unit/widget tests (bloc_test, MockCubit, MockBloc) — even when the request never says "Bloc" and it only appears in pubspec.yaml or existing code (Cubit, BlocBuilder, BlocProvider, context.read). Covers emit lifecycle (emit.forEach, no listen/Timer), restartable/droppable transformers, the project's Cubit-vs-Bloc and state conventions, immutable updates, and testing views with MockCubit. Not for Riverpod or Provider projects.
---

# Flutter Bloc

Bloc's API has been stable since 8.0, so the failures are rarely "old API". They are
**lifecycle and convention mistakes**: calling `emit` from a stream `.listen` or a `Timer`,
running a stream handler twice, mutating state lists in place, ignoring the project's
Cubit/Bloc style, and testing views against real blocs. This skill prevents those.

## Step 1 — Read the project's conventions first

Before writing code, open `pubspec.yaml` and one or two existing features (`*_bloc.dart`,
`*_cubit.dart`, the page that uses them) and note:

- **Cubit or Bloc?** If existing features use Cubit, write a Cubit. Use Bloc + events only
  when the project already does, or when you need event transformers (debounce, restartable).
- **State shape:** a `sealed class XState extends Equatable` with `final class` subclasses
  (Loading / Loaded / Failure), *or* a single state class with a `status` enum + `copyWith`.
  Copy whichever the project uses, including Equatable vs freezed.
- **Events:** `sealed class XEvent` + `final class` subclasses (Dart 3) is the norm.
- **UI:** `BlocProvider(create: ...)` in the page, `BlocBuilder` with a Dart 3 `switch` over
  sealed states, or `.when`/status checks — match the existing page.
- Dev deps: `bloc_test` and `mocktail` present? Use them in tests.

## Step 2 — `emit` rules (the #1 source of bugs)

`emit` is only valid **while its handler (or Cubit method) is still running**.

| Situation | Do | Never |
|---|---|---|
| Subscribe to a repository `Stream` in a Bloc | `await emit.forEach(stream, onData: ..., onError: ...)` | `stream.listen((x) => emit(...))`, `listen((x) => add(...))`, a `StreamSubscription` field |
| Same, but each item needs async work | `await emit.onEach(stream, onData: (x) async {...})` | nested listens |
| Debounce / delay | event transformer (Step 3) | `Timer(..., () => emit(...))` |
| Await then emit in a **Cubit** | `final r = await repo.x(); if (isClosed) return; emit(...)` | emitting after await without `isClosed` |

`emit.forEach` keeps the handler alive until the stream closes, cancels the subscription
automatically when the bloc closes, and never leaks. Register the handler that starts it
with **`transformer: restartable()`** (from `bloc_concurrency`) so a second "started"/"retry"
event cancels the previous subscription instead of creating a duplicate:

```dart
on<ChatStarted>(_onStarted, transformer: restartable());

Future<void> _onStarted(ChatStarted event, Emitter<ChatState> emit) async {
  emit(const ChatLoading());
  await emit.forEach<List<Message>>(
    _repo.watchMessages(event.roomId),
    onData: (messages) => ChatLoaded(messages),
    onError: (error, _) => ChatFailure(error.toString()),
  );
}
```

Add `bloc_concurrency` to pubspec when you use it, and say so.

## Step 3 — Concurrency and debounce

Event handlers run **concurrently** by default. Choose a transformer from `bloc_concurrency`
whenever ordering matters:

- `restartable()` — latest wins, previous handler cancelled (search, stream subscriptions).
- `droppable()` — ignore new events while busy (submit buttons, pull-to-refresh).
- `sequential()` — queue them (ordered writes).

Search-as-you-type = debounce + restartable. Either is correct:

```dart
// a) delay inside a restartable handler — cancelled handlers' emits are ignored
on<QueryChanged>(_onQueryChanged, transformer: restartable());

Future<void> _onQueryChanged(QueryChanged e, Emitter<SearchState> emit) async {
  if (e.query.isEmpty) return emit(const SearchState.empty());
  await Future<void>.delayed(const Duration(milliseconds: 300));
  emit(state.copyWith(loading: true));
  final results = await _repo.search(e.query);
  emit(SearchState.results(results));
}

// Without `transformer: restartable()` every delayed handler still runs and stale
// results can land last — the delay alone is NOT a debounce. Always pass it.

// b) explicit transformer (stream_transform)
EventTransformer<E> debounceRestartable<E>(Duration d) =>
    (events, mapper) => events.debounce(d).switchMap(mapper);
```

## Step 4 — Immutable state updates

Never mutate a list/map that lives in the current state and then emit — Equatable sees the
same object and the UI does not rebuild (and rollback breaks).

```dart
final previous = s.notes;
final updated = [for (final n in previous) n.id == id ? n.copyWith(pinned: pinned) : n];
emit(NotesLoaded(updated));             // optimistic
try {
  await _repo.setPinned(id, pinned);
} catch (_) {
  if (isClosed) return;
  emit(NotesLoaded(previous));          // rollback to the untouched old list
}
```

Not allowed on state data: `.add`, `.remove*`, `[i] = x`, `.sort()` in place.

## Step 5 — Widgets

- Provide where the bloc is created: `BlocProvider(create: (_) => XCubit(repo)..load())`.
  Use `BlocProvider.value` **only** for an instance that already exists (passing to a new
  route, tests). Never `BlocProvider.value(value: XCubit())`.
- Read vs watch: `context.read<X>()` in callbacks (`onPressed`, `onChanged`);
  `context.watch<X>()` / `context.select<X, T>(...)` or `BlocBuilder` in `build`.
  Never `context.watch` inside a callback.
- Side effects (snackbar, navigation, dialogs) go in `BlocListener` / `BlocConsumer.listener`,
  never in `builder`. Use `buildWhen` / `listenWhen` to narrow.
- Split `XPage` (provides the bloc) from `XView` (consumes it) so the view is testable.

## Step 6 — Tests

**Unit (bloc_test):**

```dart
blocTest<NotesCubit, NotesState>(
  'reverts the pin when saving fails',
  setUp: () => when(() => repo.setPinned(any(), any())).thenThrow(Exception()),
  build: () => NotesCubit(repo),
  seed: () => NotesLoaded([note]),
  act: (c) => c.togglePin(note.id),
  expect: () => [NotesLoaded([note.copyWith(pinned: true)]), NotesLoaded([note])],
);
```

Use `seed` for a starting state, `wait` for debounced events, `verify` for repo calls.

**Widget — test the View against a mock, not the real bloc:**

```dart
class MockCounterCubit extends MockCubit<int> implements CounterCubit {}

late MockCounterCubit cubit;
setUp(() => cubit = MockCounterCubit());

Widget subject() => MaterialApp(
      home: BlocProvider<CounterCubit>.value(value: cubit, child: const CounterView()),
    );

testWidgets('shows count', (t) async {
  when(() => cubit.state).thenReturn(3);
  await t.pumpWidget(subject());
  expect(find.text('3'), findsOneWidget);
});

testWidgets('tap calls increment', (t) async {
  when(() => cubit.state).thenReturn(0);
  await t.pumpWidget(subject());
  await t.tap(find.byType(FloatingActionButton));
  verify(() => cubit.increment()).called(1);   // mock state does not change
});

testWidgets('listener reacts to a state', (t) async {
  whenListen(cubit, Stream.fromIterable([9, 10]), initialState: 9);
  await t.pumpWidget(subject());
  await t.pump();
  expect(find.text('You reached 10!'), findsOneWidget);
});
```

- `MockBloc<E, S>` / `MockCubit<S>` come from `bloc_test`; stub with mocktail's `when`.
- With a mock, tapping does **not** change the displayed state — `verify` the call instead.
- Drive listeners with `whenListen`, not by tapping N times.

## Checklist before finishing

- [ ] Cubit vs Bloc, state shape and UI style match the existing features
- [ ] No `emit`/`add` from `.listen`, `Timer` or other callbacks; streams via `emit.forEach`
- [ ] Stream/search handlers use `restartable()`; `bloc_concurrency` added to pubspec if new
- [ ] State lists are copied, never mutated; rollback uses the untouched previous value
- [ ] `isClosed` checked before emitting after an `await` in Cubits
- [ ] `context.read` in callbacks; side effects in `BlocListener`
- [ ] View tests use `MockCubit`/`MockBloc` + `BlocProvider.value` + `whenListen`/`verify`

---

Maintained with eval results (with vs. without this skill) at https://github.com/lightM3/ai-skills.
