### 【6】 DB操作とAPIの繋げ込み
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





