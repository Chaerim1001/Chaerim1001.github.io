---
title: TypeORM으로 Transaction 처리하기
date: 2022-08-25
categories: [DB]

tags: [TypeORM, Transaction]
---

> 본 포스트는 아래의 환경을 기준으로 작성되었습니다. <br>
> [typeorm](https://www.npmjs.com/package/typeorm) 0.3.0 <br>
> [typeorm-tracsactional](https://www.npmjs.com/package/typeorm-transactional) 0.1.1
 {: .prompt-info }

# Transaction이란?
{: .mt-4 .mb-4 }

우선 `트랜잭션(Transaction)`이란 무엇일까요? <br>
`트랜잭션`이란 데이터베이스의 상태를 변경시키기 위해 수행하는`하나의 작업 단위`를 의미합니다.

예를 들어 게시물에 태그를 달 수 있는 상황을 생각해 볼까요? <br> (RDB를 사용하며 게시글과 태그에 대한 데이터는 각각 다른 테이블에 저장한다고 가정할게요!)
- 사용자는 게시글 하나를 작성하고 그에 맞는 태그를 입력할 것입니다.
    - 그런 뒤 사용자가 게시글 등록 버튼을 누르면 게시글과 태그 정보 각각을 저장하는 동작이 이루어져야 하죠. 
    - 이때 게시글만 저장되어서도 안되고 태그만 저장되어서도 안되며, 게시글과 태그가 일련의 작업으로 함께 저장되어야 합니다.
    - 이러한 작업 단위 하나를 `트랜잭션`이라고 부릅니다.

앞서 이야기한 것처럼 이때에는 게시글만 생성되어서도 안되고 태그만 생성되어서도 안되며, 게시글과 태그가 일련의 작업으로 함께 생성되어야 합니다. <br>
이렇게 전부 완료되거나, 하나도 완료되지 않아야 한다는 것이 트랜잭션의 중요한 특성이며 이것을 `all or nothing`이라고 합니다.

## NestJS에서 TypeORM을 통해 Transaction을 처리하는 방법
{: .mt-7 .mb-4 }

### (1) DataSource나 EntityManager 사용
TypeORM에서는 트랜잭션을 `DataSource` 혹은 `EntityManager`를 통해 만들 수 있으며, 콜백 함수를 실행하여 원하는 동작을 처리할 수 있도록 제공하고 있습니다.

``` ts
await myDataSource.manager.transaction(
    async (transactionalEntityManager) => {
        await transactionalEntityManager.save(posts)
        await transactionalEntityManager.save(tags)
    }
)
```

### (2) QueryRunner 사용
`QueryRunner`는 `Single Database Connection`을 제공하기 때문에 트랜잭션 제어가 가능합니다!
조금 더 체계적으로 직접 트랜잭션을 제어하고 싶을 때에는 `QueryRunner`를 사용할 수 있습니다.

``` ts
// queryRunner 생성
const queryRunner = dataSource.createQueryRunner()

// 새로운 queryRunner 연결
await queryRunner.connect()

// 생성한 queryRunner를 통해 쿼리문을 직접 날리는 것도 가능합니다.
await queryRunner.query("SELECT * FROM posts")

// 새로운 트랜잭션 시작
await queryRunner.startTransaction()

try {
    // 원하는 트랜잭션 동작을 정의하면 됩니다!
    await queryRunner.manager.save(new_post)
    await queryRunner.manager.save(new_tag)

    // 모든 동작이 정상적으로 수행되었을 경우 커밋을 수행합니다.
    await queryRunner.commitTransaction()
} catch (err) {
    // 동작 중간에 에러가 발생할 경우엔 롤백을 수행합니다.
    await queryRunner.rollbackTransaction()
} finally {
    // queryRunner는 생성한 뒤 반드시 release 해줘야 합니다.
    await queryRunner.release()
}
```

### (3) Transaction 데코레이터 사용
`@Transaction` 데코레이터도 사용이 가능하지만 `NestJS`에서 권장하는 방법은 아니라고 합니다.


## Example
{: .mt-7 .mb-4 }
> 지금부터는 게시글을 생성하고 게시글에 붙은 태그를 함께 저장하는 예시를 살펴봅시다. 트랜잭션 처리는 QueryRunner를 사용한 예시입니다.
{: .prompt-tip }

### (1) Entity

`Post`와 `Tag` Entity에 대한 코드입니다.

``` ts
// post.entity.ts
@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: 'varchar', length: 50 })
  title: string;

  @Column({ type: 'varchar' })
  content: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
  
  @OneToMany(() => Tag, (tag) => tag.post)
  tags: Tag[];
}
```

``` ts
// tag.entity.ts
@Entity()
export class Tag {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: 'varchar', length: 30 })
  TagName: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
  
  @ManyToOne(() => Post, (post) => post.tags)
  post: Post;
}
```

### (2) DTO 
게시물 생성에 사용될 `DTO`입니다. <br>
게시물의 title과 contents에 대한 데이터와 함께 태그에 대한 데이터도 함께 받습니다. <br>
(게시물 생성 시 태그 정보는 필수라고 가정하겠습니다.)

```ts
// create-post.dto.ts
export class CreatePostDto {
  @IsString()
  readonly title: string;
  
  @IsString()
  readonly content: string;

  @IsArray()
  @IsString({ each: true })
  readonly tag: string[];
}
```

### (3) Service
transaction을 처리하는 `Service` 코드입니다. 앞서 본 queryRunner를 사용하여 트랜잭션을 제어합니다. <br>
마지막에 finally 코드를 추가하여 생성한 queryRunner를 `release` 해주는 것을 잊지 말아야 합니다!! 🚨


```ts
// post.service.ts
 async createPost(createPostDto: CreatePostDto) {
    const { title, content, tag } = createPostDto;

    const queryRunner = this.dataSource.createQueryRunner();

    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      const post = await this.postRepository.save({
        title: title,
        content: content,
      });
      const tag = tag.map(async (tag) => {
        await this.tagRepository.save({ tagName: tag });
      });

      await queryRunner.commitTransaction();
    } catch (err) {
      await queryRunner.rollbackTransaction();
      throw err;
    } finally {
      await queryRunner.release();
    }
  }
```


<br>

> 참고 <br>
> https://docs.nestjs.com/techniques/database#transactions <br>
> https://cherrypick.co.kr/typeorm-basic-transaction
> https://orkhan.gitbook.io/typeorm/docs/transactions