![](https://velog.velcdn.com/images/fpalzntm/post/d49c76ef-65e8-4785-9ca0-d5df910b65eb/image.png)

## Riverpod 완전 가이드

### Riverpod이란?

- Riverpod은 Flutter 위젯에서 영감을 받은 새로운 방식으로 비즈니스 로직을 작성할 수 있게 해주는 상태 관리 라이브러리
- 선언적이고 반응형 프로그래밍 방식을 통해 네트워크 요청, 에러 처리, 캐싱, 자동 데이터 리페치 등 복잡한 기능들을 기본으로 처리
- 기존 `Provider` 패키지의 단점을 보완하기 위해 만들어졌으며, Provider라는 이름의 애너그램(anagram)이 이름의 유래

---

### 핵심 개념 1 — ProviderScope

- Riverpod이 동작하려면 앱의 `main` 함수에서 전체 앱을 `ProviderScope`로 감싸야 함
- 이 위젯이 모든 Provider의 상태를 저장하는 역할을 함

```dart
void main() {
  runApp(
    const ProviderScope(
      child: MyApp(),
    ),
  );
}
```

- Riverpod은 앱의 모든 상태를 루트 `ProviderScope` 위젯에 저장
- 상태가 변경될 때 앱 전체가 리빌드되는 것이 아니라, 해당 상태를 구독 중인 위젯만 선택적으로 리빌드

---

### 핵심 개념 2 — Provider의 종류

- Provider는 Riverpod의 핵심입니다.
- Provider는 슈퍼 파워를 가진 함수처럼 동작하며, 일반 함수처럼 작동하지만 캐싱, 자동 리빌드, 테스트 가능성 같은 추가 기능을 제공

| Provider 종류 | 용도 |
| --- | --- |
| `Provider` | 동기적 단순 값 노출 |
| `FutureProvider` | 비동기 API 요청 |
| `StreamProvider` | 실시간 스트림 데이터 |
| `NotifierProvider` | 상태 + 메서드(부작용) |
| `AsyncNotifierProvider` | 비동기 상태 + 메서드 |

**예시 — FutureProvider로 API 요청:**

```dart
// @riverpod 코드 생성 방식
@riverpod
Future<Joke> fetchJoke(Ref ref) async {
  final response = await dio.get('<https://api.jokes.com/random>');
  return Joke.fromJson(response.data);
}
```

- 코드 생성(`@riverpod` 어노테이션) 방식은 더 나은 문법, 향상된 가독성, 낮은 학습 곡선을 제공
- 또한 어떤 타입의 Provider를 사용할지 고민할 필요가 없어짐

---

### 핵심 개념 3 — Consumer와 ref

- 위젯에서 Provider를 읽으려면 `ref` 객체가 필요함
- `ConsumerWidget`과 `ConsumerStatefulWidget`은 각각 `StatelessWidget`/`StatefulWidget`과 `Consumer`를 합친 형태로, `build` 메서드에서 `ref`를 추가 파라미터로 받아 Provider에 접근할 수 있음.

```dart
class HomeWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch → 값 구독 + 변경 시 자동 리빌드
    final jokeAsync = ref.watch(fetchJokeProvider);

    return jokeAsync.when(
      data: (joke) => Text(joke.setup),
      loading: () => CircularProgressIndicator(),
      error: (e, st) => Text('에러: $e'),
    );
  }
}
```

**ref의 세 가지 메서드:**

- `ref.watch(provider)` — 상태를 **구독**하고 변경 시 위젯을 리빌드
- `ref.read(provider)` — 상태를 **한 번만** 읽음 (버튼 콜백 등에서 사용)
- `ref.listen(provider, callback)` — 변경을 **감지**해서 사이드 이펙트 실행

---

### 핵심 개념 4 — AsyncValue

- Riverpod에서 비동기 Provider는 `AsyncValue`를 반환하며, 이를 통해 로딩 상태, 에러 상태, 데이터 상태를 명시적으로 처리할 수 있음
- `try/catch`나 `isLoading = true/false` 같은 코드를 직접 작성할 필요가 없음

```dart
jokeAsync.when(
  data:    (joke)    => JokeCard(joke: joke),
  loading: ()        => const CircularProgressIndicator(),
  error:   (e, st)   => ErrorWidget(error: e),
);
```

---

### 핵심 개념 5 — Provider 간 의존성

- Provider들은 `ref` 객체를 통해 다른 Provider를 구독할 수 있음
- `ref.watch`를 사용하면 구독 중인 값이 변경될 때 해당 Provider도 자동으로 업데이트 됨

```dart
@riverpod
List<Todo> filteredTodos(Ref ref) {
  final todos  = ref.watch(todosProvider);   // 다른 Provider 구독
  final filter = ref.watch(filterProvider);  // 다른 Provider 구독

  return switch (filter) {
    Filter.all       => todos,
    Filter.completed => todos.where((t) => t.completed).toList(),
    Filter.active    => todos.where((t) => !t.completed).toList(),
  };
}
```

---

### 핵심 개념 6 — NotifierProvider (상태 변경)

- 읽기 전용 Provider와 달리, `Notifier`는 상태를 외부에서 변경할 수 있는 메서드를 포함함.

```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;  // 초기값

  void increment() => state++;
  void reset()      => state = 0;
}

// 위젯에서 사용
ref.read(counterProvider.notifier).increment();
```

---

### 핵심 개념 7 — autoDispose와 생명주기

- 코드 생성 방식을 사용할 경우 Provider는 기본적으로 `autoDispose` 상태
- 즉, 리스너가 없어지면 자동으로 Provider가 삭제
- 이 기본 설정은 Riverpod의 철학과 잘 맞다. 만약 이 동작을 끄고 싶다면 `@Riverpod(keepAlive: true)` 어노테이션을 사용하면 됨

```dart
@riverpod               // autoDispose = true (기본값)
String example1(Ref ref) => 'foo';

@Riverpod(keepAlive: true)  // autoDispose 비활성화
String example2(Ref ref) => 'foo';
```

---

### 핵심 개념 8 — Family (파라미터 전달)

- 코드 생성 방식에서는 `family` 모디파이어 없이도 Provider 함수에 일반 파라미터를 바로 전달할 수 있어, 이전 방식보다 훨씬 직관적

```dart
// 코드 생성 방식 (권장)
@riverpod
Future<User> fetchUser(Ref ref, {required int userId}) async {
  return await api.getUser(userId);
}

// 사용
ref.watch(fetchUserProvider(userId: 42));
```

---

### 좋은 습관 (Best Practices)

- Provider는 외부(위젯 등)에서 초기화해서는 안 되고, 스스로 초기화해야 함.
- 외부에서 초기화하면 레이스 컨디션이나 예상치 못한 동작이 발생할 수 있음
- Provider는 반드시 최상위 `final` 변수로 선언
- 클래스 내부의 인스턴스 변수로 생성하면 메모리 누수와 예상치 못한 동작이 발생

---

# 요약

- Riverpod은 `ProviderScope → Provider 정의 → ConsumerWidget에서 ref.watch`라는 세 가지 핵심 흐름으로 이루어짐
- 비동기 상태는 `AsyncValue`가 자동으로 래핑해주기 때문에 로딩/에러 처리가 매우 간결해지고, Provider 간 의존성도 `ref.watch`만으로 깔끔하게 연결됨
