## 【6】 DB操作とAPIの繋げ込み
[talk]
DB接続の準備と、TODOアプリのためのDBモデルの準備を行いました。
このセクションから、DBに接続するRead/Writeの処理と、これをAPIに繋げて動作を確認してみましょう。


### 1. C: Create
最初はデータが存在しないので、 POST /tasks から書いていきましょう。
1-1. 以下のファイルを編集して、保存をします。 


api/cruds/task.py

――――――――――――――――――――――――――

    from sqlalchemy.ext.asyncio import AsyncSession


    import api.models.task as task_model
    import api.schemas.task as task_schema




    async def create_task(
        db: AsyncSession, task_create: task_schema.TaskCreate
    ) -> task_model.Task:


      task = task_model.Task(**task_create.dict()
      db.add(task)
      await db.commit()
      await db.refresh(task)
      return task
      
――――――――――――――――――――――――――

**コードの説明**

**処理フロー**
やっていることの大まかな流れを箇条書きで書き下してみます。


1. 引数としてスキーマ **task_create: task_schema.TaskCreate** を受け取ります
2. これをDBモデルである **task_model.Tas**k に変換する
3. DBにコミットする
4. DB上のデータを元に、Taskインスタンス task を更新する（この場合、作成したレコードの id を取得する）
5. 作成したDBモデルを返却する


[talk]
これが大まかな流れです。ここで注目したいのは、関数定義が async def となっていること、
それから db.commit() と db.refresh(task) に await が付いていることです。
以下、説明をしていきます。


async def は関数が非同期処理を行うことができる、 「コルーチン関数」であるということを表します。


### コルーチンとは
コルーチン（英: co-routine）とはプログラミングの構造の一種。
サブルーチンがエントリーからリターンまでを一つの処理単位とするのに対し、コルーチンはいったん処理を中断した後、
続きから処理を再開できる。

[コルーチン - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%AB%E3%83%BC%E3%83%81%E3%83%B3)

[Python で学ぶ、コルーチンと非同期処理 - Qiita](https://qiita.com/koshigoe/items/054383a89bd51d099f10#:~:text=%E3%82%B3%E3%83%AB%E3%83%BC%E3%83%81%E3%83%B3%EF%BC%88%E8%8B%B1%3A%20co%2Droutine,%E5%8B%95%E4%BD%9C%E3%82%92%E8%A1%8C%E3%81%86%E3%81%93%E3%81%A8%E3%81%AB%E3%82%88%E3%82%8B%E3%80%82)



[talk]
詳細は上記のURLに詳しい説明があるので、ご覧になってください。
ここでは、コルーチンは処理中断後でも続きから再開できるプログラムの構造だと理解してしていただければと思います。
もう一つ、awaitについて説明します。


**await**は、非同期処理、ここではDBへの接続（IO処理）が発生するため「待ち時間が発生するような処理をしますよ」ということを
示しています。
これによって、pythonはこのコルーチンの処理からいったん離れてから、イベントループ内で別のコルーチンの処理を行うことができます。
これが非同期・並行処理の肝要になります。


上記のCRUDs定義を利用するルーターは、以下のように書き直します。


1-2. 以下のファイルを編集して、保存をします。
api/routers/task.py
――――――――――――――――――――――――――

    from fastapi import APIRouter, Depends
    from sqlalchemy.ext.asyncio import AsyncSession

    import api.cruds.task as task_crud
    from api.db import get_db


    @router.post("/tasks", response_model=task_schema.TaskCreateResponse)
    async def create_task(
        task_body: task_schema.TaskCreate, db: AsyncSession = Depends(get_db)
    ):
        return await task_crud.create_task(db, task_body)
――――――――――――――――――――――――――

**コードの説明**

先程の **create_task()** と同様、ルーターのパスオペレーション関数もコルーチンとして定義されています。
ですので、**create_task()** からの返却値も **await** を使って返却します。


また、**db: AsyncSession = Depends(get_db)** にも注目してみましょう。
Depends は引数に関数を取り、 DI（Dependency Injection、依存性注入） を行う機構です。
DB接続部分にDIを利用することにより、ビジネスロジックとDBが密結合になることを防ぐことが利点として挙げられます。
例えば、テストの時に、DBインスタンスの中身を外部から上書きできるので、**get_db** と異なるテスト用の接続先を置換するといったことが、
コードに触れることなく、可能になります。

[参考][PythonでのDependency Injection 依存性の注入 - Qiita](https://qiita.com/mkgask/items/d984f7f4d94cc39d8e3c)



[talk] 

プログラミングの文脈で言われている「依存」とは、特定のクラスの内部で使用される値が、 
外部の何か（変数、定数、クラスのインスタンスなど）に依存している状態を指します。 
ここでは、依存性注入の詳細な説明は省きますが、簡潔にいうと、その依存している先を外部から注入出来るようにすることで、 
依存していたクラス互いのクラスの結合度が下がり、より汎用性の高いクラスにすることができるメリットがあります。 




**DBモデルとレスポンススキーマの変換**


リクエストボディの **task_schema.TaskCreate** と、レスポンスモデルの **task_schema.TaskCreateResponse** については、  

リクエストに対して id だけを付与して返却する必要があります。 
 

1-3. 以下のファイルを編集して、保存をします。 

api/schemas/task.py 

―――――――――――――――――――――――――― 

    class TaskBase(BaseModel): 
        title: Optional[str] = Field(None, example=“買い物に行く") 


    class TaskCreate(TaskBase): 
        pass 


    class TaskCreateResponse(TaskCreate): 
        id: int 



     class Config: 
        orm_mode = True 

―――――――――――――――――――――――――― 

**コードの説明** 

**orm_mode = True** は、このレスポンススキーマ TaskCreateResponse が、暗黙的にORMを受け取り、レスポンススキーマに変換することを意味します。 


### 2. R: Read

前項でTask の作成が可能になったので、次に Task をリストで受け取る Read エンドポイントを作成しましょう。 


ToDoアプリでは Task に対して Done モデルが定義されていますが、これらを別々に Read で取得するのは面倒な為、 
これらをjoinして、ToDoタスクにDoneフラグが付加された状態のリストを取得できるエンドポイントとしましょう。 

2-1. 以下のファイルを編集して、保存をします。 


api/cruds/task.py 

―――――――――――――――――――――――――― 

    async def get_tasks_with_done(db: AsyncSession) -> List[Tuple[int, str, bool]]: 
        result: Result = await ( 
            db.execute( 
                select( 
                    task_model.Task.id, 
                    task_model.Task.title, 
                    task_model.Done.id.isnot(None).label("done"), 
                ).outerjoin(task_model.Done) 
            ) 
        ) 

        return result.all() 

―――――――――――――――――――――――――― 

**コードの説明** 

get_tasks_with_done() も create_task() と同様コルーチンなので、 async def で定義され、 await を使って Result を取得します。 
コルーチンの返却値として result.all() コールによって、初めてすべてのDBレコードを取得します。 
select() で必要なフィールドを指定し、 .outerjoin() によってメインのDBモデルに対してjoinしたいモデルを指定しています。 



また、 dones テーブルは tasks テーブルと同じIDを持ち、ToDoタスクが完了しているときだけレコードが存在していると説明しました。 

**task_model.Done.id.isnot(None).label("done")** によって、 Done.id が存在するときは **done=True** とし、 

存在しないときは done=False として、joinしたレコードを返却します。 

 

上記のCRUDs定義を利用するルーターは、以下のコードを追記します。 

構造はCreateとのものと同等です。 

 

2-2. 以下のファイルを編集して、保存をします。 

 

api/routers/task.py 

―――――――――――――――――――――――――― 

    #追記 
    @router.get("/tasks", response_model=List[task_schema.Task]) 
    async def list_tasks(db: AsyncSession = Depends(get_db)): 
        return await task_crud.get_tasks_with_done(db) 

――――――――――――――――――――――――――

### 3. U: Update 

Update も Create とほぼ同等ですが、最初にリクエストしているのが存在している Task に対してなのかをチェックし、 
存在した場合は更新、存在しない場合は404エラーを返却するAPIにします。 



3-1. 以下のファイルを編集して、保存をします。 

 

api/cruds/task.py 

※2つの関数を定義します 

―――――――――――――――――――――――――― 

    async def get_task(db: AsyncSession, task_id: int) -> Optional[task_model.Task]: 
        result: Result = await db.execute( 
            select(task_model.Task).filter(task_model.Task.id == task_id) 
        ) 

        task: Optional[Tuple[task_model.Task]] = result.first() 
        return task[0] if task is not None else None  # 要素が一つであってもtupleで返却されるので１つ目の要素を取り出す 

    async def update_task( 
        db: AsyncSession, task_create: task_schema.TaskCreate, original: task_model.Task 
    ) -> task_model.Task: 

        original.title = task_create.title 
        db.add(original) 
        await db.commit() 
        await db.refresh(original) 
        return original 
―――――――――――――――――――――――――― 

**コードの説明** 

get_task() では、 .filter() を使って SELECT ~ WHERE として対象を絞り込んでいます。 
また、Result は select() で指定する要素が１つであってもtupleで返却されてしまいます。 
その為、tupleではなく値として取り出すためには処理が必要になります。 
 

update_task() は create_task() とほとんど見た目が同じです。  
original としてDBモデルを受け取り、これの中身を更新して返却しています。 


上記のCRUDs定義を利用するルーターは、以下のコードを追記します。 

構造はCreateとのものと同等です。 

 

3-2. 以下のファイルを編集して、保存をします。 

api/routers/task.py 

―――――――――――――――――――――――――― 

    #追記 
    @router.put("/tasks/{task_id}", response_model=task_schema.TaskCreateResponse) 
    async def update_task( 
        task_id: int, task_body: task_schema.TaskCreate, db: AsyncSession = Depends(get_db) 
    ): 
        task = await task_crud.get_task(db, task_id=task_id) 
        if task is None: 
            raise HTTPException(status_code=404, detail="Task not found")  

        return await task_crud.update_task(db, task_body, original=task) 

―――――――――――――――――――――――――― 


**コードの説明** 

HTTPException は任意のHTTPステータスコードを引数に取ることができる Exception クラスです。 

今回は 404 Not Found を指定して raise します。 

 

### 4. D: Delete 

Delete のインターフェイスも Update とほぼ同等です。まず get_task を実行してから、delete_task を実行します。 

 
4-1. 以下のファイルを編集して、保存をします。 
 

api/cruds/task.py 

―――――――――――――――――――――――――― 

    #追記 
    async def delete_task(db: AsyncSession, original: task_model.Task) -> None: 
        await db.delete(original) 
        await db.commit() 

―――――――――――――――――――――――――― 


上記のCRUDs定義を利用するルーターは、以下のコードを追記します。 

4-2. 以下のファイルを編集して、保存をします。 


api/routers/task.py 

―――――――――――――――――――――――――― 

    #追記 
    @router.delete("/tasks/{task_id}", response_model=None) 
    async def delete_task(task_id: int, db: AsyncSession = Depends(get_db)): 
        task = await task_crud.get_task(db, task_id=task_id) 
        if task is None: 
            raise HTTPException(status_code=404, detail="Task not found") 

        return await task_crud.delete_task(db, original=task) 

―――――――――――――――――――――――――― 


**コードの説明** 

UPDATEと同様の為、説明は省きます。 


### 5. Doneリソースの定義 

Taskリソースと同様、Doneリソースも定義していきましょう。 


5-1. 以下のファイルを編集して、保存をします。 
 

api/cruds/done.py 
―――――――――――――――――――――――――― 

    async def get_done(db: AsyncSession, task_id: int) -> Optional[task_model.Done]: 
        result: Result = await db.execute( 
            select(task_model.Done).filter(task_model.Done.id == task_id) 
        ) 
        done: Optional[Tuple[task_model.Done]] = result.first() 
        return done[0] if done is not None else None  # 要素が一つであってもtupleで返却されるので１つ目の要素を取り出す 

    async def create_done(db: AsyncSession, task_id: int) -> task_model.Done: 
        done = task_model.Done(id=task_id) 
        db.add(done) 
        await db.commit() 
        await db.refresh(done) 
        return done 

    async def delete_done(db: AsyncSession, original: task_model.Done) -> None: 
        await db.delete(original) 
        await db.commit() 
    
―――――――――――――――――――――――――― 


上記のCRUDs定義を利用するルーターは、以下のコードを追記します。 


5-2. 以下のファイルを編集して、保存をします。 

api/routers/done.py 

―――――――――――――――――――――――――― 

    @router.put("/tasks/{task_id}/done", response_model=done_schema.DoneResponse) 
    async def mark_task_as_done(task_id: int, db: AsyncSession = Depends(get_db)): 
        done = await done_crud.get_done(db, task_id=task_id) 
        if done is not None: 
            raise HTTPException(status_code=400, detail="Done already exists") 
        return await done_crud.create_done(db, task_id) 

    @router.delete("/tasks/{task_id}/done", response_model=None) 
    async def unmark_task_as_done(task_id: int, db: AsyncSession = Depends(get_db)): 
        done = await done_crud.get_done(db, task_id=task_id) 
        if done is None: 
            raise HTTPException(status_code=404, detail="Done not found") 

        return await done_crud.delete_done(db, original=done) 

―――――――――――――――――――――――――― 


レスポンススキーマが必要の為、 api/schemas/done.py も同時に作成していきます。 


5-3. 以下のファイルを編集して、保存をします。 
 

api/schemas/done.py 

―――――――――――――――――――――――――― 

    class DoneResponse(BaseModel): 
        id: int 

        class Config: 
            orm_mode = True 

―――――――――――――――――――――――――― 


**コードの説明** 

今までの定義と同様の為、説明は省きます。 

後ほど、swaggerのパートで説明しますが、条件に応じて下記の挙動になることに注目してください。 


・完了フラグが立っていないとき 

　・PUT: 完了フラグが立つ 

　・DELETE: フラグがないので 404 エラーを返す 

・完了フラグが立っているとき 

　・PUT: 既にフラグが経っているので 400 エラーを返す 

　・DELETE: 完了フラグを消す 
 
 6. 最終的なディレクトリの構成 

以上で、ToDoアプリに必要なファイルは全て定義できました。 

最終的には以下のようなファイル構成になっているはずです。 

―――――――――――――――――――――――――― 

    api 

    ├── __init__.py 
    ├── db.py 
    ├── main.py 
    ├── migrate_db.py 
    ├── cruds 
    │   ├── __init__.py 
    │   ├── done.py 
    │   └── task.py 
    ├── models 
    │   ├── __init__.py 
    │   └── task.py 
    ├── routers 
    │   ├── __init__.py 
    │   ├── done.py 
    │   └── task.py 
    └── schemas 
        ├── __init__.py 
        ├── done.py 
        └── task.py 

―――――――――――――――――――――――――― 

[talk] 

ここまでFastAPIで、ToDoアプリを作成しましたが、CRUDアプリとして正しい挙動なのか気になります。 
FastAPIは冒頭で説明したように、SwaggerUIとの連携が強力です。 
CRUDの動作確認は、そのSwaggerから確認できます。 
次は実際にToDoアプリのCRUD機能ができているか、Swaggerで、確認しましょう。 



## 【7】Swaggerによるアプリの機能確認 

※ この箇所は、春樹さん or 石崎さんが担当。 

[talk] 

ここまで、TODOアプリの製造とDB接続処理、APIの繋げ込みを行いました。 
このセクションから、CRUDアプリの各機能が動作するか、Swaggerを使用して検証します。 
FastAPIはSwaggerとの連携が強力です。実際に使用して確認していきましょう！ 



### 1. C: Createの動作確認 

Swagger UIから POST /tasks エンドポイントにアクセスしてみましょう。 

「Execute」 を押す度に、 id がインクリメントされて結果が返却されることがわかります。 

テストとして、5回ぐらい「Execute」 を押してみましょう。 

 

### 2. R: Readの動作確認 

Create を叩いた回数だけTODOタスクが作成されており、すべてがリストで返却されます。 

 

### 3. U: Updateの動作確認 

task_id=1 のタイトルを変更してみましょう。前項のReadインタフェースから、変更後の結果が取得できることを確認します。 

 

### 4. D: Deleteの動作確認 

task_id=2 を削除してみましょう。もう一度、実行すると既に削除されているので404エラーが返ります。 
同じく、Readインタフェースから削除がされていることを確認しましょう。 



### 5. D: Doneの動作確認 

Doneリソースの Mark Task As Done を実行後、Taskリソースの Read インターフェイスで、 doneフラグが操作できたことが確認できます。 

完了フラグの条件で、400、404エラーを返すことが確認できます。 

 
 

 
 

[talk] 

以上で、TODOアプリの全ての挙動がSwaggerUI上で確認することができました。 
このことからFastAPIとswaggerの連携が強力であることがお分かりになったかと思います。 
それ以外にも、SwaggerUIによる自動ドキュメント生成など便利な機能があるので、活用してみてください。 
以上で、本日のミートアップの内容は終了ですが、この後、設定していただいた環境を元に戻しますので、少々お付き合いください。 



## 【8】ローカル環境のcleanUP 

[talk] 

docker-composeで作られた、コンテナ、イメージ、ボリューム等、全てを一括消去するコマンドがありますので、今回はそれを使用します。 

 

### 1. ローカル環境のdocker-composeを一括削除 

新しくターミナルを開き、以下コマンドで 一括削除を行います。 



8-1. 以下コマンドを実行し、下記の出力とプロンプトが戻ることを確認します。 

$ docker-compose down --rmi all --volumes --remove-orphans 

―――――――――――――――――――――――――― 

    # 実行結果の例 
    Stopping my-php-app_app_1 ... done 
    Stopping my-php-app_db_1  ... done 
    Removing my-php-app_app_1 ... done 
    Removing my-php-app_db_1  ... done 

―――――――――――――――――――――――――― 

2. コンテナの停止確認 

8-2. 以下コマンドを実行し、下記の出力とプロンプトが戻ることを確認します。 

$ docker-compose ps 

―――――――――――――――――――――――――― 

    Name   Command   State   Ports 
      ------------------------------ 

―――――――――――――――――――――――――― 

上記の状態であれば、コンテナが停止しています。 



8-2. 以下コマンドを実行し、下記の出力とプロンプトが戻ることを確認します。 

$ docker-compose images 

―――――――――――――――――――――――――― 

        Container           Repository       Tag       Image Id       Size   
    ------------------------------------------------------------------------ 
    fastapi_db_1         mysql              8.0      d1dc36cf8d9e   519.4 MB 
    fastapi_demo-app_1   fastapi_demo-app   latest   cd033663f8f9   1.073 GB 

―――――――――――――――――――――――――― 



※上記のように、IMAGE IDが存在する場合、イメージを削除します。 

$ docker-compose images 

docker rmi -f [IMAGE ID] 


### 2. プロジェクトフォルダの削除 

最後にプロジェクトフォルダを削除します。 
 

以上で、本日のハンズオンは終了となります。皆様ありがとうございました！ 
