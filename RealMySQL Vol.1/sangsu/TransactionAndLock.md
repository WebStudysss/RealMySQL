# 트랜잭션

- 데이터 작업의 **원자성(Atomicity)** 을 보장한다.
- 즉, 하나의 작업 단위 안에 포함된 모든 연산이 **모두 성공하거나, 모두 실패**해야 한다.
- MySQL에서는 InnoDB가 트랜잭션을 지원하며, MyISAM은 지원하지않음

## 트랜잭션을 사용할 때 주의사항

당신이 운영하고있는 서비스에서 하기 항목으로 진행되는 로직이 존재할 때 이것이 최선이라 볼 수 있는가?

```java
1. 처리 시작
=> 데이터베이스 커넥션 생성
=> 트랜잭션 시작
2. 사용자의 로그인 여부 확인
3. 사용자의 글쓰기 내용의 오류 여부 확인
4. 첨부로 업로드된 파일 확인 및 저장
5. 사용자의 입력 내용을 DBMS에 저장
6. 첨부 파일 정보를 DBMS에 저장
7. 저장된 내용 또는 기타 정보를 DBMS에서 조회
8. 게시물 등록에 대한 알림 메일 발송
9. 알림 메일 발송 이력을 DBMS에 저장
<= 트랜잭션 종료(COMMIT)
<= 데이터베이스 커넥션 반납
10. 처리 완료

```

 우리가 보통 상황에서 위와 같은 로직을 직접 구현한다고 생각해보자.
주로 사용하는 Java Spring으로 간략하게 예시를 작성해보았다. 로그인 여부 확인은 사용자 검증으로 대체, 트랜잭션이 너무 길게 잡혀있어서 DB측 문제를 야기하는 상황이라고 가정해보자

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/posts")
public class PostController{
  
  private final PostService postService;
  private final CustomFileSystem fileSystem;
  private final NotiService notiService;
  private final LogService logService;

  @PostMapping
  public CommonResult create(PostRequest request) {
    postService.create(request);  
    return postService.getPosts(cursor);
  }

}

@Service
@RequiredArgsConstructor
public class PostService{

  private final PostRepository postRepository;
  private final UserService userServic;  

  @Transactional
  public PostResponse create(PostRequest request){
	  
	  //사용자 상태 가져오기
	  User user = userService.findUserById(request.getUserId());
	  
	  //유저 검증
	  user.isActiveUser(ActiveType.POST);
	  
	  //글쓰기 내용의 오류 여부 확인
	  if(!Post.isPostUsable(request)){
		  throw new IllegalArgumentException();
	  }
	  
	  //첨부로 업로드된 파일 확인 및 저장
	  String filePath = request.getFilePath();
	  if(CustomFileSystem.fileValid(filePath)){
	    throw new IllegalArgumentException();
	  }
	  String storedFilePath = CustomFileSystem.store(filePath);
	  
	  //사용자의 입력 내용을 DBMS에 저장
	  Post post = Post.posting(request);
	  //post.save();
	  
	  //첨부 파일 정보를 DBMS에 저장
	  FileEntity file = fileSystem.storeFile(storedFilePath);
	  //file.save();
	  
	  //저장된 내용 또는 기타 정보를 DBMS에서 조회
	  List<User> followUserList = userService.getFollowUser();
	  
	  //게시물 등록에 대한 알림 메일 발송
	  notiService.postNoticeToFollowUsers(followUserList);
	  
	  //알림 메일 발송 이력을 DBMS에 저장
	  logService.notiLog(followUserList);
	  //log.save();
	  
  }
  
}

```

위의 코드는 메서드 시작부터 트랜잭션이 적용되는데, DB접근이 필요 없는 파일을 저장하는 부분에서 트랜잭션이 너무 오래 걸린다고 가정해보면, 첨부 파일을 저장할 때 많은 시간이 소요된다면, 해당 시간에는 트랜잭션이 돌지 않도록 하는 것도 트랜잭션 과부화를 줄일 수 있는 방법이 될 것 같다.

```java
@Service
@RequiredArgsConstructor
public class PostService {
  private final TransactionTemplate tx;          // 또는 @Transactional 메서드
  private final PostRepository postRepository;
  private final ApplicationEventPublisher events;

  public PostResponse create(PostRequest request) {
    // (1) 트랜잭션 바깥: 비DB 유효성/파일 임시처리 등
    validate(request);                      // 런타임 예외
    String tempPath = CustomFileSystem.storeToTemp(request.getFilePath());

    // (2) 트랜잭션 안: 오직 DB만
    Post post = tx.execute(status -> {
      Post entity = Post.posting(request);
      postRepository.save(entity);
      // 커밋 후 처리할 이벤트만 발행(아웃박스 사용 시 아웃박스 레코드 저장)
      events.publishEvent(new PostCreatedEvent(entity.getId(), tempPath));
      return entity;
    });

    // (3) 커밋 이후: 부작용은 여기서(혹은 AFTER_COMMIT 리스너/아웃박스 워커)
    // 이 줄을 직접 실행하는 대신, AFTER_COMMIT에 맡기는 걸 권장
    return PostResponse.of(post);
  }
}

```

최적화 방식에는 다른 방법도 많고, 대규모 시스템 설계에서는 아웃박스 패턴을 가장 권장한다고 한다.
아웃박스 참고 문서 : https://medium.com/@greg.shiny82/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%94%EB%84%90-%EC%95%84%EC%9B%83%EB%B0%95%EC%8A%A4-%ED%8C%A8%ED%84%B4%EC%9D%98-%EC%8B%A4%EC%A0%9C-%EA%B5%AC%ED%98%84-%EC%82%AC%EB%A1%80-29cm-0f822fc23edb

# MySQL 엔진의 잠금

MySQL에서 지원하는 잠금은 크게 MySQL 엔진 레벨, 스토리지 엔진 레벨로 구분이 가능

## 글로벌 락

SELECT를 제외한 대부분의 DDL 문장이나 DML 문장은 락이 걸려서 동작하지 못한다. 활용 예시로는 MyISAM이나 MEMORY 스토리지 엔진의 일관된 백업을 받아야 할 때 사용한다.

## 테이블 락

개별 테이블 단위로 설정되는 잠금으로, MyISAM이나 MEMORY 테이블에 데이터를 변경하는 쿼리를 실행할 때 발생한다.

InnoDB에서는 스키마를 변경하는 DDL의 경우에만 영향을 미친다

## 네임드 락

임의의 문자열에 대해 잠금을 설정할 수 있다.

## 메타데이터 락

데이터베이스 객체에 대한 이름이나 구조를 변경하는 경우에 획득하는 락

# InnoDB 스토리지 엔진 잠금

## 레코드 락

레코드 자체만을 잠그는 것을 레코드 락이라고 하며, InnoDB 레코드 락의 특징으로는 레코드가 아니라 인덱스의 레코드를 잠그는 것이다. 인덱스가 하나도 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.

## 갭 락

레코드가 아닌, 레코드와 인접한 레코드 사이의 간격을 잠그는 것을 의미한다. 레코드와 레코드 사이의 간격에 새로운 레코드가 생성되는 것을 제어

## 넥스트 키 락

MySQL 격리수주는 REPEATABLE READ 격리수준이 관정되는데, 레코드 락과 갭 락을 같이 사용하는 경우를 뜻한다.

## 자동 증가 락

AUTO_INCREMENT를 사용할 때 번호가 겹치지 않고 사용되게 하기 위해 하드웨어 단의 뮤텍스를 활용해 락을 건다.

## 인덱스와 잠금

InnoDB는 데이터 변경이 있을 때 인덱스에 락을 건다고 했다. 이는 인덱스를 적절히 설정하지 못하면, 테이블 전체 레코드에 대해 풀스캔 및 락이 걸리게 되기에 적절한 인덱스가 중요하다.

## MySQL의 트랜잭션 격리 수준

|  | DIRTY READ | NON-REPEATABLE READ | PHANTOM READ |
| --- | --- | --- | --- |
| READ UNCOMMITED | 발생 | 발생 | 발생 |
| READ COMMITED | 없음 | 발생 | 발생 |
| REPEATABLE READ | 없음 | 없음 | 발생(InnoDB는 발생x) |
| SERIALIZABLE | 없음 | 없음 | 없음 |
- READ UNCOMMITED : 트랜잭션 내의 커밋되지 않은 데이터도 읽힌다
- READ COMMITED : 트랜잭션 간 커밋이 완료된 데이터만 읽힌다
- REPEATABLE READ : 트랜잭션 내에서 레코드 멱등성을 보장한다
- SERIALIZABLE : 커밋이 완료가 되어야 데이터가 읽힌다

- DIRTY READ : 커밋되지 않은 데이터도 읽힌다
- NON-REPEATABLE READ : 한 트랜잭션 내에서 특정 데이터를 읽을 때 다른 트랜잭션이 해당 데이터 변경 후 커밋 완료했을 시 데이터 값이 변경됨(레코드 멱등성x)
- PHANTOM READ : 다른 트랜잭션에서 행이 추가되면 추가된 행이 같이 읽힌다(행 수 멱등성x)
