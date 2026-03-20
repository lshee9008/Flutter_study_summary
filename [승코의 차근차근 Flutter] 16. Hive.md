# Hive: 빠르고 가벼운 로컬 데이터베이스

- 사용자 설정이나 간단한 데이터를 로컬에 저장해야 할 때
- 이때 SharedPreferences나 SQLite를 많이 떠올린다.
- 속도와 편의성 면에서 가장 사랑받는 패키지 중 하나가 Hive임.

# 1. 하이브(Hive)란 무엇인가?

- Hive는 순수 Dart로 작성된 가볍고 엄청나게 빠른 Key-Value(키-값) 기반의 NoSQL 데이터베이스
- 복잡한 테이블이나 쿼리 없이, 마치 딕셔너리(Map)에 데이터를 넣고 빼듯이 직관적으로 사용 가능

### 왜 Hive를 사용해야 할까?(장점)

- 놀라운 속도
    - 네이티브 의존성 없이 순수 Dart로 작성되어 다른 로컬 DB패키지들보다 읽기/쓰기 속도가 압도적으로 빠름
- 크로스 플랫폼 지원
    - 모바일, 데스크톱, 웹 등 플러터가 지원하는 모든 플랫폼에서 동일하게 동작
- 강력한 보안
    - 내장된 AES-256 암호화를 통해 민감한 데이터를 안전하게 보호할 수 있음
- 간편한 사용법
    - 복잡한 SQL 문법을 배울 필요 없이, Box라는 개념을 통해 쉽게 데이터를 다룸.

# 2. Hive 설치 및 기본 설정

- 먼저 pubspec.yml 파일에 필요한 패키지들을 추가

```yaml
dependencies:
  flutter:
    sdk: flutter
  hive: ^2.2.3          # Hive 코어 패키지
  hive_flutter: ^1.1.0  # Flutter 환경에 맞는 확장 기능

dev_dependencies:
  hive_generator: ^2.0.1 # TypeAdapter 자동 생성을 위한 패키지
  build_runner: ^2.4.0   # 코드 생성을 실행하는 도구
```

- pub.dev에서 최신 버전을 확인 권장

# 3. 기본 사용법 (CRUD 구현하기)

- Hive에서는 데이터를 저장하는 공간을 “Box(박스)”라고 부름
- 데이터를 읽고 쓰려면 먼저 Box를 열어야 함
1. 초기화 및 Box 열기
    - main.dart 파일에서 앱이 실행되기 전에 Hive을 초기화하고 Box를 연다.
        
        ```dart
        import 'package:flutter/material.dart';
        import 'package:hive_flutter/hive_flutter.dart';
        
        void main() async {
          // Flutter 바인딩 초기화
          WidgetsFlutterBinding.ensureInitialized();
          
          // Hive 초기화 (로컬 디렉토리 설정 등)
          await Hive.initFlutter();
          
          // 'myBox'라는 이름의 Box 열기
          await Hive.openBox('myBox');
          
          runApp(const MyApp());
        }
        ```
        
2. 데이터 쓰기, 읽기, 수정, 삭제(CRUD)
    - 사용법은 Dart의 Map을 다루는 것만큼 간단.
        
        ```dart
        // 열어둔 Box 가져오기
        var box = Hive.box('myBox');
        
        // 1. Create & Update (데이터 쓰기 및 수정)
        // key가 존재하지 않으면 새로 생성하고, 있으면 덮어씁니다.
        box.put('name', 'Gemini');
        box.put('age', 25);
        
        // 2. Read (데이터 읽기)
        // key에 해당하는 값이 없으면 null 또는 기본값을 반환합니다.
        String name = box.get('name', defaultValue: 'Unknown');
        int age = box.get('age');
        
        // 3. Delete (데이터 삭제)
        box.delete('name');
        ```
        

# 4. 커스텀 객체 저장하기 (TypeAdapter)

- 기본적인 String, int 외에 우리가 직접 만든 클래스(예: User, Todo)를 저장하려면 TypeAdapter가 필요
- hive_generator를 사용하여 이 과정을 자동화할 수 있음
1. 모델 클래스 작성(user.dart)
    - @HiveType과 @HiveField 어노테이션을 사용하여 데이터 모델을 정의
        
        ```dart
        import 'package:hive/hive.dart';
        
        // 생성될 파일 이름을 지정합니다.
        part 'user.g.dart';
        
        @HiveType(typeId: 0) // typeId는 0~223 사이의 고유한 숫자여야 합니다.
        class User {
          @HiveField(0)
          String name;
        
          @HiveField(1)
          int age;
        
          User({required this.name, required this.age});
        }
        ```
        
2. Build Runner 실행
    - 터미널에 아래 명령어를 입력하여 user.g.dart (어댑터 파일)를 자동 생성
        
        ```bash
        flutter pub run build_runner build
        ```
        
3. 어댑터 등록 및 사용
    - 다시 main.dart로 돌아와서 Box를 열기 전에 생성된 어댑터를 등록해 줌
        
        ```dart
        void main() async {
          WidgetsFlutterBinding.ensureInitialized();
          await Hive.initFlutter();
          
          // 생성된 어댑터 등록
          Hive.registerAdapter(UserAdapter());
          
          // User 객체 전용 Box 열기
          await Hive.openBox<User>('userBox');
          
          runApp(const MyApp());
        }
        ```
        
    - 이제 객체 단위로 데이터를 저장하고 불러올 수 있음
        
        ```dart
        var userBox = Hive.box<User>('userBox');
        userBox.put('user1', User(name: 'Gemini', age: 25));
        ```
