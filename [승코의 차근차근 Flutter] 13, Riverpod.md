# Riverpod이 필요한 이유 (Provider와의 차이점)

- Riverpod은 Provider 2.0이라기보다는 완전히 새로운 패러다임

| **특징** | **Provider** | **Riverpod** |
| --- | --- | --- |
| **에러 시점** | 런타임 (앱 실행 중 `ProviderNotFoundException` 발생) | **컴파일 타임** (코드 작성 시점에 오류 발견) |
| **의존성** | `BuildContext`에 강하게 의존 | **Context 불필요** (어디서든 상태 접근 가능) |
| **결합** | 동일한 타입의 Provider 여러 개 생성 어려움 | 동일 타입의 Provider 무제한 생성 가능 |
| **테스트** | 위젯 트리 구성 필요 (복잡함) | 순수 Dart 객체처럼 테스트 가능 (간편함) |

# Riverpod의 핵심 개념 3가지

1. Provider (공급자)
    - 상태(데이터)를 캡슐화하고 제공하는 객체
    - 전역 변수처럼 선언되지만, 상태는 불변(Immutable)하게 관리 됨.
2. Provider Scope (스코프)
    - 앱의 최상위에 위치하여 모든 Provider의 상태를 저장하는 저장소
        
        ```dart
        void main() {
        	runApp(
        		//앱을 ProviderScope로 감사야 함
        		ProviderScope(
        			child: MyApp(),
        		),
        	);
        }
        ```
        
3. Ref (참조자)
    - Provider와 상호작용하기 위한 도구임
    - 다른 Provider를 읽거나, 위젯에서 상태를 구독할 때 사용
    - ref.watch(provider) : 상태가 변경되면 위젯을 다시 빌드함. (주로 UI에서 사용)
    - ref.read(provider) : 상태를 한 번만 읽거나, 함수(메서드)를 실행할 때 사용함. (주로 onPressed 등 이벤트에서 사용)
    - ref.listen(provider, (prev, next) {…}) : 상태 변화를 감지하여 특정 액션을 수행함. (다이얼로그 표시, 페이지 이동 등)

# Riverpod 2.0의 주요 Provider 종류

- 최신 Riverpod(2.0 이상)에서는 Code Generation(코드 생성) 을 적극 권장함. 이를 사용하면 문법이 훨씬 간결해짐
1. 기본 데이터 제공 (@riverpod 함수형)
    - 단순히 값을 익기만 하거나, 의존성 주입이 필요할 때 사용
        
        ```dart
        @riverpod
        String helloWorld(HelloWorldRef ref) {
        	return 'Hello Riverpod';
        }
        ```
        
2. NotifierProvider (@riverpod 클래스형)
    - 동기적(Synchronous)인 상태를 관리하고 수정할 때 사용함. (예: 카운터, 테마 변경)
        
        ```dart
        @riverpod
        class Counter extends _&Counter {
        	@override
        	int build() => 0; //초기값 설정
        	
        	void increment() => state++;
        }
        ```
        
3. AsyncNotifierProvider
    - 비동기(Asynchronous) 데이터(API 호출 등)를 관리할 때 가장 강력한 도구임. 로딩, 에러, 데이터 상태를 자동으로 처리해주는 AsyncValue를 반환함.
        
        ```dart
        @riverpod
        class TodoList extends _$TodoList {
        	@overrid
        	Future<List<Todo>> build() async {
        		return fetchTodos();
        	}
        	
        	Future<void> addTodo(Todo todo) async {
        		state = const AsyncValue.loading();
        		state = await AsyncValue.guard(() async {
        			await api.addTodo(todo);
        			return fetchTodos();
        		});
        	}
        }
        ```
        
4. UI에서 사용하기 (ConsumerWidget)
    - Riverpod을 사용할 때는 StatelessWidget 대신 ConsumerWidget을 상속받습니다. build 메서드에 WidgetRef 가 추가됨.
    - 기본 상태 읽기
        
        ```dart
        class CounterApp extends ConsumerWidget {
        	@override
        	widget build(BuildContext context, WidgetRef ref) {
        		// 1. 상태 구독 (값이 바뀌면 build 재실행)
        		final count = ref.watch(counterProvider);
        		
        		return Scaffold(
        			body: Center(child: Text('$count')),
        			floatingActionButton: FloatingActionButton(
        				// 2. 메서드 실행 (read 사용)
        				onPressed: () => ref.read(counterProvider.notifier).increment(),
        				child: Icon(Icons.add),
        			),
        		);
        	}
        }
        ```
        
    - 비동기 데이터 처리(AsyncValue)
        - AsyncNotifier를 사용할 때는 when을 사용하여 로딩, 에러, 성공 상태를 우아하게 처리할 수 있음
            
            ```dart
            class TodoPage extends ConsumerWidget {
            	@override
            	Widget build(BuildContext context, WidgetRef ref) {
            		final todoAsync = ref.watch(todoListProvider);
            		
            		return todoAsync.when(
            			data: (todos) => ListView(children: ...), // 데이터 로드 성공 시
            			loading: () => CircularProgressIndicator(), // 로딩 중
            			error: (err, stack) => Text('Error: $err'), // 에러 발생 시
            		);
            	}
            }
            ```
            
5. Riverpod Generator 설정 (권장)
- 최신 Riverpod을 100% 활용하려면 코드 생성기를 설정해야 함
    
    ```yaml
    dependencies:
      flutter_riverpod: ^2.x.x
      riverpod_annotation: ^2.x.x
    
    dev_dependencies:
      build_runner: ^2.x.x
      riverpod_generator: ^2.x.x
    ```
    

# 요약: 언제 무엇을 써야 할까?

1. 단순 읽기 전용 값: Provider (함수형 @riverpod)
2. 변경 간능한 동기 상태 (카운터, 필터): NotifierProvider (클래스형 @riverpod)
3. API 호출 및 비동기 상태: AsyncNotifierProvider (Future 반환하는 클래스형 @riverpod)
4. 스트림 (웹소켓 등): StreamProvider

# 핵심 장점 한 줄 요약

- Riverpod은 비동기 데이터 처리(로딩/에러/성공)를 AsyncValue 하나로 완벽하게 해결하며, 컴파일 타임에 안전한 코드를 작성하게 해줌.
