# 왜 기존 Provider 대신 Riverpod인가?

- Riverpod은 Provider 패키지의 작성자인 Remi Rousselet이 Provider의 근본적인 한계를 극복하기 위해 아예 처음부터 다시 만든 라이브러리
- 컴파일 타임 안정성
    - 기존 Provider는 위젯 트리 구조에 의존했기 때문에, 트리에 없는 Provider를 호출하면 런타임 에러(ProviderNotFoundException)가 발생
    - Riverpod은 전역에 선언되므로 위젯 트리와 무관하게 어디서든 안전하게 접근 할 수 있음.
- 동일한 타입 다중 선언 가능
    - 기존 Provider는 타입 기반으로 값을 찾기 때문에 Provider<String>을 여러 개 만들면 충돌이 났음.
    - Riverpod은 객체 인스턴스(변수명)로 식별하므로 제한이 없음
- 보일러플레이트 감소 (Code Generation)
    - 최근 Riverpod은 @riverpod 어노테이션을 활용한 코드 생성 방식을 권장함.
    - 덕분에 개발자가 Provider 타입을 일일이 고민할 필요 없이, 함수나 클래스만 작성하면 알아서 최적의 코드가 생성

# 2. 핵심 개념 심화 및 실전 문법

1. ProviderScope: 상태 저장소
    - Riverpod의 Provider들은 전역 변수처럼 선언되지만, 실제 상태(데이터)는 전역에 저장되지 않고 루트에 있는 ProviderScope 내부에 안전하게 격리되어 저장
2. Consumer 위젯의 종류
    - 상태를 읽기 위해 사용하는 ref 객체를 얻는 방법은 위젯의 종류에 따라 다르다.
        - ConsumerWidget
            - StatelessWidget을 대체한다.
            - build 메서드에 WidgetRef가 추가됨
        - ConsumerStatefulWidget & ConsumerState
            - StatefulWidget을 대체함.
            - 이 방식의 장점은 build 메서드뿐만 아니라 initState, dispose 등 생명주기 메서드 안에서도 ref 속성에 바로 접근할 수 있다는 점
        
        ```dart
        class NewsScreen extends ConsumerStatefulWidget {
          @override
          ConsumerState<NewsScreen> createState() => _NewsScreenState();
        }
        
        class _NewsScreenState extends ConsumerState<NewsScreen> {
          @override
          void initState() {
            super.initState();
            // initState 안에서도 ref.read 사용 가능!
            ref.read(analyticsProvider).logScreenView('NewsScreen');
          }
        
          @override
          Widget build(BuildContext context) {
            // build에서는 ref.watch 사용
            final newsList = ref.watch(newsListProvider);
            return Scaffold(...);
          }
        }
        ```
        
3. ref.watch, ref.read, ref.listen 완벽 가이드
    - ref.watch(provider): 반응형 구독, 상태가 바뀌면 build 함수를 다시 실행, 반드시 build 메서드나 다른 Provider 내부에서만 사용해야 함.
    - ref.read(provider)
        - 일회성 읽기. 버튼의 onPressed 콜백과 같이 사용자 액션에 의해 트리거되는 이벤트 핸들러에서 사용(build 안에서 사용 금지)
    - ref.listen(provider, (previous, next) { … })
        - 상태 변화에 따른 사이드 이펙트(부수 효과)를 처리함
        - 예를 들어, 에러 상태로 변경되었을 때 스낵바(SnackBar)를 띄우거나, 로그인 성공 시 화면을 이동(Routing)할 때 사용, build 메서드 안에서 등록
        
        ```dart
        @override
        Widget build(BuildContext context, WidgetRef ref) {
          // 에러 발생 시 스낵바 띄우기
          ref.listen<AsyncValue<User>>(userProvider, (previous, next) {
            if (next.hasError) {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(content: Text('로그인 실패: ${next.error}')),
              );
            }
          });
        
          return ...;
        }
        ```
        
4. 비동기 처리의 꽃, AsyncValue와 패턴 매칭
    - API에서 데이터를 가져오는 작업은 대기(Loading) → 성공(Data) 또는 실패(Error)의 과정을 거침
    - 기존에서는 isLoading 같은 상태 변수를 따로 관리해야 했지만, AsyncValue는 이 모든 상태를 캡슐화함.
    - Dart 3의 패턴 매칭(Pattern Matching)을 결합하면 더욱 우아하게 작성할 수 있음
        
        ```dart
        Widget build(BuildContext context, WidgetRef ref) {
          final asyncNews = ref.watch(fetchNewsProvider);
        
          // Dart 3의 switch 문법과 결합한 최신 방식
          return switch (asyncNews) {
            AsyncData(:final value) => ListView.builder(
                itemCount: value.length,
                itemBuilder: (_, i) => Text(value[i].title),
              ),
            AsyncError(:final error) => Center(child: Text('에러 발생: $error')),
            _ => const Center(child: CircularProgressIndicator()), // 로딩 상태
          };
        }
        ```
        
5. 상태 변경의 주역: Notifier와 AsyncNotifier
    - 단순히 데이터를 읽어오는 것을 넘어, 사용자의 상호작용으로 데이터를 수정(Post/Put/Delete)해야 할 때 사용
    - 특히 백엔드(FastAPI 등)와 통신하며 상태를 업데이트할 때는 비동기 작업을 다루는 AsyncNotifier가 필수적
    
    ```dart
    @riverpod
    class TodoList extends _$TodoList {
      // 초기 데이터 로드 (GET 요청)
      @override
      Future<List<Todo>> build() async {
        return await api.getTodos();
      }
    
      // 데이터 추가 (POST 요청)
      Future<void> addTodo(String title) async {
        // 1. 상태를 강제로 로딩 상태로 변경하여 UI에 스피너 표시
        state = const AsyncValue.loading();
        
        // 2. 비동기 작업 수행 (백엔드에 추가) 및 결과에 따라 상태 업데이트
        state = await AsyncValue.guard(() async {
          await api.postTodo(title);
          // 성공하면 목록을 다시 불러와서 상태 갱신
          return await api.getTodos(); 
        });
      }
    }
    ```
    
    - Tip: AsyncValue.guard는 내부에서 발생하는 try-catch를 자동으로 처리하여 결과를 AsyncData나 AsyncError로 만들어주는 매우 유용한 헬퍼 메서드이다.
6. 캐싱, 생명주기 관리, 그리고 ref.onDispose
    - 코드 생성 방식(@riverpod)을 쓰면 기본적으로 화면에서 안 쓰이는 데이터는 즉시 메모리에서 날아감(autoDispose)
    - 하지만 메모리에서 해제될 때 WebSocket을 닫거나, Timer를 취소하는 등의 정리 작업이 필요할 수 있다. 이때 ref.onDispose를 사용
    
    ```dart
    @riverpod
    Stream<String> voiceRecognition(Ref ref) {
      final controller = VoiceController();
      controller.startListening();
    
      // 해당 Provider를 구독하는 위젯이 사라지면 실행됨 (리소스 낭비 방지)
      ref.onDispose(() {
        controller.stopListening();
        controller.dispose();
      });
    
      return controller.stream;
    }
    ```
    

### 7. 성능 최적화: select의 활용

- 객체의 특정 속성만 변경되었을 때 화면을 다시 그리고 싶다면 select를 사용
- 이렇게 하면 불필요한 위젯 리빌드를 막아 앱의 퍼포먼스가 크게 향상
    
    ```dart
    // User 객체 전체가 아니라, user.name이 바뀔 때만 이 위젯을 리빌드함
    final userName = ref.watch(userProvider.select((user) => user.name));
    ```
    

# 정리: 왜 개발자들은 Riverpod에 영광할까?

1. 의존성 주입이 너무 쉽다.: 다른 클래스(예: ApiClient, Repository)를 매개변수로 넘겨줄 필요 없이, Provider 안에서 ref.watch(apiClientProvider) 한 줄이면 끝남.
2. 안전한 비동기 처리: FutureBuilder의 콜백 지옥이나 예기치 않은 로딩 스피너 버그 없이, 데이터를 동기식 코드처럼 직관적으로 다룰 수 잇음
3. 확장성: 작은 토이 프로젝트부터 복잡한 아키텍처를 가진 대규모 프로덕션 앱까지 무리 없이 대응할 수 있는 견고함을 제공
