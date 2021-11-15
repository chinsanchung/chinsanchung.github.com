---
title: '원티드 프리온보딩 코스 세 번째 과제 후기'
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - review
toc: true
toc_sticky: true
toc_labe: 목차
description: 프리온보딩 백엔드 코스의 세 번째 과제를 수행하면서 겪은 경험을 작성합니다.
tags:
  - 위코드
  - 원티드
  - freshcode
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 세 번째 과제를 수행한 경험을 정리하고자 이 글을 작성했습니다. [Github repository 링크](https://github.com/wanted-wecode-subjects/redbrick-subject)에서 작성한 코드를 확인하실 수 있습니다.

## 1. 개요

이번 프로젝트는 [레드브릭](https://wizschool.notion.site/wizschool/Redbrick-the-new-land-of-opportunity-f449bf0490f6468a8cb04ae3c96ed98b)에서 제시한 과제입니다.

**레드브릭**은 누구나 쉽게 자신만의 소프트웨어를 창작할 수 있도록, 소프트웨어 창작 대중화를 꿈꿉니다.

주제는 게시글 API 를 만드는 것으로, 세부적인 사항은 아래와 같습니다.

### [필수 요구 사항]

- 회원가입
- 게임 제작
  - 프로젝트는 실시간으로 반영이 되어야 합니다
    - 프로젝트 수정 중 의도치 않은 사이트 종료 시에도 작업 내역은 보존되어야 합니다
- 게임 출시하기
  - **프로젝트 당 퍼블리싱 할 수 있는 개수는 하나**입니다.
  - 퍼블리싱한 게임은 수정할 수 있어야 하며, 수정 후 재출시시 기존에 퍼블리싱된 게임도 수정됩니다
  - 출시하는 게임은 다른 사용자들도 볼 수 있으며, 사용자들의 **조회수 / 좋아요 등을 기록**할 수 있어야 합니다
  - '게임 혹은 사용자 **검색**'을 통해서 찾을 수 있어야 합니다

### [개발 요구사항]

```jsx
- 참고 - 문제 1,2번은 필수 문제이며, 3번은 선택입니다
문제 1. '회원가입'부터 '게임 출시'까지 필요한 테이블을 설계하세요

문제 2. 다음에 필요한 API를 설계하세요

	1) 게임 제작 API
	2) 조회수 수정, 좋아요 API
	3) 게임/사용자 검색 API

- option -
문제 3.
 (1) 프로젝트 실시간 반영을 위한 Architecture를 설계하세요 ( 그림이 있다면 좋습니다 )
 (2) 위의 Architecture를 토대로 기능을 구현하세요
```

### 개발 환경

- 언어: TypeScript
- 프레임워크: NestJs
- 데이터베이스: SQLite3
- 라이브러리: typeorm, passport, passport-jwt, bcrypt, class-validator, class-transformer, cache-manager, @nestjs/schedule

## 회고

이번 과제에서 담당한 기능은 프로젝트의 CRUD, 게임의 검색, 프로젝트를 출시하여 게임 생성, 그리고 프로젝트의 유닛 테스트입니다.

### typeorm 익숙해지기

mongoose 를 이용해 mongoDB 만을 사용해왔기에, sqlite3 과 typeorm 을 이용한 데이터와의 통신 방식에 익숙해질 필요가 있었습니다.

#### 엔티티에서의 관계 설정

typeorm 을 이용해 엔티티를 정의하면서 다른 테이블과의 연결을 설정하는 방법을 배웠습니다.

mongoose 에서는 스키마의 user_id 필드를 `user_id: {type: Schema.Types.ObjectId, ref:'users'}`으로 선언하고, 쿼리문에서 이 필드를 활용하면 해당 컬렉션의 내용을 불러올 수 있었습니다. 하지만 sql, typeorm 에서는 엔티티에서 관계를 명시하는 작업이 필요합니다.

```typescript
@Entity()
export class Project extends CoreEntity {
  @Column()
  title: string;

  @Column({ nullable: true })
  code: string;

  @Column({ type: Boolean, default: false })
  isPublished: boolean;

  @ManyToOne((_type) => User, (user) => user.projects, {
    eager: true,
    onDelete: 'CASCADE',
  })
  user: User;
}
```

여기서 주목할 부분은 `@ManyToOne`입니다. 이 데코레이터를 사용한 이유는 유저 한 사람이 여러 프로젝트를 사용할 수 있고, 따라서 유저와 프로젝트의 관계는 1:N 이기 때문입니다. 주의할 점은 프로젝트뿐만 아니라 유저의 엔티티에서도 프로젝트와의 관계를 설정해야 합니다.

```typescript
@Entity()
export class User extends CoreEntity {
  // ...
  @OneToMany((_type) => Project, (project) => project.user, {
    eager: false,
    cascade: true,
  })
  projects: Project[];
}
```

설정에서 eager, onDelete, cascade 의 의미가 무엇인지 알아보겠습니다.

- eager: 값을 true 으로 하여 eager 관계를 설정하면, typeorm 의 `find`메소드를 사용했을 때 해당 영역의 데이터를 자동으로 불러옵니다. 여기서는 프로젝트에서 find 를 실행할 때 유저의 정보를 불러오게 됩니다. 다만, [공식 문서](https://typeorm.io/#/eager-and-lazy-relations)에 따르면 이 설정은 한 쪽에서만 수행해야지, 두 테이블 모두에 eager 를 true 로 하는 것은 권장하지 않는다고 합니다.
- onDelete: 레퍼런스가 되는 객체를 삭제할 때 외래키를 어떻게 다룰 지를 정합니다. 프로젝트의 유저 컬럼을 선언하면서 "CASCADE"로 설정했는데, 그럴 경우 유저를 삭제할 경우 그 유저가 작성한 프로젝트도 모두 삭제합니다.
- cascade: true 로 하면, 관련된 객체를 데이터베이스에 추가하고 업데이트할 수 있습니다. `("insert" | "update" | "remove" | "soft-remove" | "recover")[]` 라는 배열 형태의 옵션으로 어떤 기능을 수행할 때 cascade 를 수행할 지를 결정할 수 있습니다.

#### softDelete

이번 과제에서는 데이터를 삭제할 때 실제 데이터를 삭제하거나, 아니면 특정 컬럼을 이용해 삭제했다는 것을 표시해주는 방법 중 후자를 택했습니다. 그러기 위해선 그에 대한 컬럼을 선언하고, `delete`메소드가 아닌 `softDelete`를 사용해야 합니다.

우선 엔티티에서의 설정입니다. 공통으로 사용하는 id, createdAt, updatedAt, deletedAt 컬럼을 따로 CoreEntity 에 선언했습니다. 주목할 점은 `@DeleteDateColumn`입니다. 이를 통해 삭제를 수행했을 때 이 컬럼의 값을 자동으로 바꿉니다.

```typescript
export class CoreEntity {
  // ...
  @DeleteDateColumn()
  deletedAt: Date;
}
```

마지막으로 typeorm 에서 지원하는 `softDelete` 메소드로 소프트 삭제를 수행합니다.

#### createQueryBuilder

검색 기능을 구현할 때 "게임의 제목" 또는 "유저의 이름"으로 검색하기 위해선 `find`메소드보다는 입맛대로 쿼리를 작성할 수 있는 `createQueryBuilder`메소드를 사용하기로 했습니다. 마치 mongoDB 의 aggregate 와 비슷한 역할을 수행하는데, 이 메소드로 관계를 설정하고, where 조건을 추가할 수 있습니다.

```typescript
const data = await this.gameRepository
  .createQueryBuilder('game')
  .innerJoin('game.user', 'user')
  .where(`game.title like :keyword`, { keyword: `%${keyword}%` })
  .orWhere(`user.nickname like :keyword`, { keyword: `%${keyword}%` })
  .limit(limit)
  .offset(offset)
  .getMany();
```

1. 먼저 `innerJoin`으로 게임의 user_id 컬럼으로부터 유저 정보를 불러옵니다.
2. 그 다음 `where`, `orWhere`메소드로 조건문을 실행합니다.
3. 마지막으로 페이지네이션을 위해 limit(최대 개수), offset(페이지 수만큼 생략할 항목)을 설정합니다.

### 순환 참조

프로젝트의 정보를 이용해 게임을 생성하는 "게임 퍼블리싱"을 작성하려면 프로젝트의 서비스에서 게임의 서비스로부터 기능을 불러올 필요가 있습니다. NestJs 에서는 모듈의 `exports`를 통해 게임 서비스를 외부에서 활용할 수 있도록 할 수 있습니다.

```typescript
// game.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([GameRepository]), ProjectsModule],
  controllers: [GameController],
  providers: [GameService],
  exports: [GameService],
})
export class GameModule {}
```

exports 를 이용해 내보낸 게임의 서비스를 프로젝트에서 사용하려면 `import`에서 불러와야 합니다.

```typescript
// project.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Project]), GameModule, MemoryCacheModule],
  controllers: [ProjectsController],
  providers: [ProjectsService],
  exports: [ProjectsService],
})
export class ProjectsModule {}
```

하지만 게임과 프로젝트 모두 각자의 서비스를 이용하고 있는 상황입니다. 이럴 경우 [순환 종속성](https://docs.nestjs.kr/fundamentals/circular-dependency)이 발생합니다. 지금과 같이 단순하게 모듈을 가져오는 것만으로는 에러가 발생합니다.

순환 종석성을 위해서 `forwardRef` 유틸리티 함수를 이용해 모듈을 연결해야 합니다. 주의할 점은 두 모듈 모두 이 함수로 연결해야 한다는 것입니다.

```typescript
// game.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([GameRepository]),
    forwardRef(() => ProjectsModule),
  ],
  // ...
})

// project.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([Project]),
    forwardRef(() => GameModule),
  ],
  // ...
})
```

### 트랜젝션

게임을 퍼블리싱을 하면서 트랜젝션을 이용했습니다. 트랜젝션이란 [쿼리 하나가 실패하면, 데이터베이스 시스템은 전체 트랜잭션 또는 실패한 쿼리를 롤백](https://ko.wikipedia.org/wiki/%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4_%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98)하는 것으로, 문제가 생겨 퍼블리싱을 취소해야 할 때 수행했던 작업을 처음으로 되돌리기 위해 사용합니다.

트랜젝션을 사용한 이유는 프로젝트와 게임 데이터에 접근해 조회, 수정을 수행하는 복잡한 과정에서 작업 도중에 에러가 발생하면 그 전에 했던 것을 취소해야 하는데, 트랜젝션으로 되돌리는 것이 직접 일일이 수행하는 것보다 편리하기 때문입니다.

[NestJs 공식 문서](https://docs.nestjs.kr/techniques/database#transactions)에서 작성한 typeorm 을 이용한 트랜젝션 이용법을 참고했습니다.

1. 우선 앱 모듈에서 트랜젝션을 이용할 것임을 선언해야 합니다. 앱 모듈의 생성자에 `Connection`을 파라미터로 넣습니다.

```typescript
export class AppModule {
  constructor(private connection: Connection) {}
}
```

2. 그 다음 트랜젝션을 수행할 곳에서 `Connection`을 선언합니다.

```typescript
export class ProjectsController {
  constructor(
    private readonly projectsService: ProjectsService,
    @Inject(forwardRef(() => GameService))
    private readonly gameService: GameService,
    private connection: Connection
  ) {}
}
```

3. 트랜젝션을 수행합니다. queryRunner 생성 -> connect() -> startTransaction -> commitTransaction() 또는 rollbackTransaction() -> release() 의 순으로 작성합니다.

```typescript
export class ProjectsController {
  @Post('/publish/:id')
  async publishProject(
    @Param('id') id: string,
    @Body() publishProjectDto: PublishProjectDto,
    @GetUser() user: User
  ): Promise<IPublishResponseMessage> {
    const queryRunner = this.connection.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();
    try {
      const project = await this.projectsService.findOne(+id);
      // user가 project 작성자인지 확인
      // ...
      if (!project.isPublished) {
        // 게임이 없는 경우 게임 생성
        // ...
      } else {
        // 게임이 존재하는 경우 게임 수정 또는 게임이 삭제된 적이 있는 경우
        // ...
      }
      await queryRunner.commitTransaction();
      return { message: 'publish complete' };
    } catch (error) {
      await queryRunner.rollbackTransaction();
      return { message: 'publish fail' };
    } finally {
      await queryRunner.release();
    }
  }
}
```

퍼블리싱 과정에서 에러가 발생하면 `rollbackTransaction`메소드로 수행한 과정을 취소하고 롤백합니다. 만약 성공한다면, `commitTransaction`메소드로 수행한 결과를 데이터베이스에 저장합니다.

## 프로젝트를 마치며

프로젝트를 수행하면서 typeorm 의 작동 방식을, 또한 순환 참조 이외에도 여러 기능들을 작성하고 점검하면서 nestJS 프레임워크를 연습할 수 있었습니다. nestJs 로 과제를 수행하면서 점점 익숙해지고 있는데, 앞으로 남은 프로젝트도 nestJs 를 사용해서 제작할 예정입니다. 팀 과제를 수행하면서 엄격한 정의와 구조를 지향하는 nestJs 프레임워크를 사용하는 것이 Express 로 제작하는 것보다 더 편리하기 떄문입니다.
