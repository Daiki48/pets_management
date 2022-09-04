# 使用するライブラリ

- crossterm

TUI ライブラリ

- serde

`JSON` ファイルを扱うためのライブラリ

- chrono

作成日を扱うためのライブラリ

- rand

新しいランダムデータを扱うためのライブラリ

# 構造体の定義

データベースとして利用する `JSON` ファイルの定数と、ペットに関する構造体を定義します。

```rust
const DB_PATH: &str = "./data/db.json";

#[derive(Serialize, Deserialize, Clone)]
struct Pet {
    id: usize,
    name: String,
    category: String,
    age: usize,
    created_at: DateTime<Utc>,
}
```

データベースファイルを扱う際、`I/O` エラーが発生する可能性があります。全てのエラーに対応することは難しいですが、いくつかの内部エラータイプを持たせます。

`db.json` は、`Pet` 構造体のリストを JSON で表現しただけのものです。

```json
[
    {
        "id": 1,
        "name": "Chip",
        "category": "cats",
        "age": 4,
        "created_at": "2020-09-01T12:00:00Z"
    },
    ...
]
```

また、入力イベントのためのデータ構造も必要です。

```rust
enum Event<I> {
    Input(I),
    Tick,
}
```

イベントとは、ユーザーからの入力、または単なる刻みのことです。ティックレート（例えば 200 ミリ秒）を定義し、そのティックレート内に入力イベントが起きなければ、 `Tick` を発することになります。そうで無い場合は、入力が発生します。

最後に、アプリケーションのどこにいるのかを簡単に判断出来るように、メニュー構造の `enum` を定義しておきます。

```rust
#[derive(Copy, Clone, Debug)]
enum MenuItem {
    Home,
    Pets,
}

impl From<MenuItem> for usize {
    fn from(input: MenuItem) -> usize {
        match input {
            MenuItem::Home => 0,
            MenuItem::Pets => 1,
        }
    }
}
```

今は、 `Home` と `Pets` の 2 つのページしかないので、それらを `usize` に変換する方法を実装しました。これにより、TUI の `Tabs` コンポーネント内の enum を使用して、現在選択されているタブをメニューで強調表示することが出来るようになりました。

構造体などの初期設定を終えたら、TUI と Crossterm をセットアップして、画面へのレンダリングとユーザーイベントへの反応を開始します。

# レンダリングと入力

まず、端末を `raw` (または non-canonical) モードに設定し、ユーザーによる Enter を待つ必要がないようにして、入力に反応するようにします。

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    enable_raw_mode().expect("can run in raw mode");
...
```

次に、インプットハンドラとレンダリングループの間で通信するための `mpsc` (multiproducer, single consumer) チャンネルを設定します。

```rust
let (tx, rx) = mpsc::channel();
    let tick_rate = Duration::from_millis(200);
    thread::spawn(move || {
        let mut last_tick = Instant::now();
        loop {
            let timeout = tick_rate
                .checked_sub(last_tick.elapsed())
                .unwrap_or_else(|| Duration::from_secs(0));

            if event::poll(timeout).expect("poll works") {
                if let CEvent::Key(key) = event::read().expect("can read events") {
                    tx.send(Event::Input(key)).expect("can send events");
                }
            }

            if last_tick.elapsed() >= tick_rate {
                if let Ok(_) = tx.send(Event::Tick) {
                    last_tick = Instant::now();
                }
            }
        }
    });
```

チャンネルを作成した後、前述のティックレートを定義し、スレッドを生成してください。ここで、入力ループを行います。このスニペットは TUI と Crossterm を使用して入力ループをセットアップするための推奨方法です。

アイデアは、次のティック（タイムアウト）を計算し、`event::poll` を使ってその時間までイベントを持ち、もしイベントがあれば、ユーザーが押したキーでチャンネルを通してその入力イベントを送信することです。

このタイムアウト時間内にユーザーイベントが発生しなかった場合は、単に `Tick` イベントを送信して最初からやり直します。 `tick_rate` を使えば、アプリケーションの応答性を調整することが出来ます。しかし、これを低く設定しすぎると、このループがたくさん実行され、リソースを無駄遣いしてしまいます。

アプリケーションのレンダリングにはメインスレッドが必要なため、このロジックは別のスレッドで生成されます。この方法では、入力ループがレンダリングをブロックすることはありません。

Crossterm バックエンドで TUI ターミナルをセットアップするためには、いくつかのステップが必要です。

```rust
    let stdout = io::stdout();
    let backend = CrosstermBackend::new(stdout);
    let mut terminal = Terminal::new(backend)?;
    terminal.clear()?;
```

`stdout` を使用して `CrosstermBackend` を定義し、TUI Terminal でそれを使用し、最初にそれをクリアして、全てが動作することを暗黙のうちにチェックしました。このいずれかが失敗すると、単にパニックになり、アプリケーションは停止します。イベントレンダリングの変更を何も取得していないので、ここで反応する良い方法はありません。

これで、入力ループと描画可能な端末を作るためのお決まりの設定は終わりました。

次に、最初の UI 要素であるタブベースのメニューを作成してみます。

# TUI でのウィジェットの描画

メニューを作成する前に、レンダリングループを実装する必要があります。レンダリングループは、繰り返しごとに `terminal.draw()` を呼び出します。

```rust
    loop {
        terminal.draw(|rect| {
            let size = rect.size();
            let chunks = Layout::default()
                .direction(Direction::Vertical)
                .margin(2)
                .constraints(
                    [
                        Constraint::Length(3),
                        Constraint::Min(2),
                        Constraint::Length(3),
                    ]
                    .as_ref(),
                )
                .split(size);
...
```

draw 関数にクロージャを提供し、クロージャは rect を受け取ります。これは、TUI で使われる短形のレイアウトプリミティブで、ウィジェットがレンダリングされる場所を定義するために使われます。

次は、レイアウトのチャンクを定義することです。ここでは、3 つのボックスからなる伝統的な縦型レイアウトを採用しています。

1. メニュー
2. コンテンツ
3. フッター

TUI の `Layout` を使用して、方向といくつかの制約を設定して、これを定義することができます。これらの制約は、レイアウトの異なるパーツをどのように構成すべきかを定義します。この例では、メニュー部分の長さを `3` (3 行分) 、コンテンツ部分の長さを `2` 以上、フッター部分の長さを `3` として定義しています。これは、このアプリをフルスクリーンで実行した場合、メニューとフッターは常に 3 行の高さで一定となり、コンテンツ部分は残りのサイズに拡大されることを意味します。

これらの制約は強力なシステムであり、後で見るパーセントや比率を設定するオプションも用意されています。最後に、レイアウトをレイアウトチャンクに分割します。

このレイアウトで最も単純な UI コンポーネントは、偽の著作権を持つ静的なフッターです。したがって、ウィジェットの作成とレンダリングの方法を知るために、このフッターから始めてみます。

```rust:main.rs
	let copyright = Paragraph::new("pet-CLI 2020 - all rights reserved")
			.style(Style::default().fg(Color::LightCyan))
			.alignment(Alignment::Center)
			.block(
					Block::default()
							.borders(Borders::ALL)
							.style(Style::default().fg(Color::White))
							.title("Copyright")
							.border_type(BorderType::Plain),
			);

	rect.render_widget(copyright, chunks[2]);
```

TUI に数多く存在する既存のウィジェットの 1 つである、パラグラフ・ウィジェットを使用します。ハードコードされたテキストでパラグラフを定義し、デフォルトのスタイルに `.fg` を使用して異なる前景色を使用することでスタイルを設定し、アライメントをセンターに設定し、ブロックを定義しています。

このブロックは、他のすべてのウィジェットがレンダリングするために使用出来る「基本」ウィジェットであるため、重要です。ブロックは、タイトルとボックス内にレンダリングするコンテンツの周囲にオプションでボーダーを置くことが出来る領域を定義します。

今回は、 `Copyright` というタイトルのボックスを作成し、フルボーダーで 3 ボックスの縦長レイアウトを実現しました。

次に、`draw` メソッドの `rect` を使用して `render_widget` を呼び出し、レイアウトの 3 番目の部分（下部）である `chunks[2]` に段落をレンダリングします。

TUI でウィジェットをレンダリングする方法は以上です。とてもシンプルです。

# タブのメニューを構築

これでようやく、タブベースのメニューに取り組むことができます。幸いなことに、TUI にはタブ・ウィジェットが最初から備わっているので、動作させるために多くのことを行う必要はありません。どのメニュー・アイテムがアクティブであるかという状態を管理するために、レンダリング・ループの前に 2 行を追加する必要があります。

```rust
...
    let menu_titles = vec!["Home", "Pets", "Add", "Delete", "Quit"];
    let mut active_menu_item = MenuItem::Home;
...
loop {
    terminal.draw(|rect| {
...
```

最初の行では、ハードコードされたメニューのタイトルを定義し、 `active_menu_item` には、現在選択されているメニューの項目が格納され、最初は `Home` に設定されています。

しかし、2 つのページしかないので、これは `Home` か `Pets` にしか設定されませんが、この方法が基本的なトップレベルのルーティングにどのように使われるかは、おそらく想像がつくでしょう。

メニューを表示するには、まずメニューラベルを保持するための `Span` (HTML の span タグのようなもの) 要素のリストを作成し、それらを `Tabs` ウィジェット内に配置する必要があります。

```rust
let menu = menu_titles
		.iter()
		.map(|t| {
				let (first, rest) = t.split_at(1);
				Spans::from(vec![
						Span::styled(
								first,
								Style::default()
										.fg(Color::Yellow)
										.add_modifier(Modifier::UNDERLINED),
						),
						Span::styled(rest, Style::default().fg(Color::White)),
				])
		})
		.collect();

let tabs = Tabs::new(menu)
		.select(active_menu_item.into())
		.block(Block::default().title("Menu").borders(Borders::ALL))
		.style(Style::default().fg(Color::White))
		.highlight_style(Style::default().fg(Color::Yellow))
		.divider(Span::raw("|"));

rect.render_widget(tabs, chunks[0]);
```

ハードコードされたメニュー・ラベルを繰り返し処理し、それぞれのラベルについて、最初の文字で文字列を分割しています。こうすることで、最初の文字に異なる色と下線をつけ、ユーザーにこの文字がこのメニュー項目を有効にするために入力する必要がある文字であることを示すことができます。

この操作の結果は Span 要素であり、これは単に Span のリストです。これは、任意にスタイル設定された複数のテキスト片にほかなりません。これらのスパンは、`Tabs` ウィジェットに格納されます。

`active_menu_item` で `.select()` を呼び、通常のスタイルとは異なるハイライトスタイルを設定しています。
つまり、メニュー項目が選択されている場合は完全に黄色で表示され、選択されていないものは最初の文字だけが黄色で、残りは白になります。また、分割を定義し、再びボーダーとタイトルを持つ基本ブロックを設定し、スタイルの一貫性を保ちます。

コピーライトと同じように、タブメニューをレイアウトの最初の部分に `chunks[0]` を使ってレンダリングします。

これで、3 つのレイアウト要素のうち 2 つが完成しました。しかし、ここではステートフル要素を定義し、 `active_menu_item` を、どの要素がアクティブであるかのストレージとして使用しています。これはどのように変化するのでしょうか？

この謎を解くために、次に入力処理についてみてみましょう。

# TUI での入力操作

レンダリング `loop {}` の中で、 `terminal.draw()` の呼び出しが完了した後、別のコード片を追加しています。

つまり、常に現在の状態を最初にレンダリングし、それから新しい入力に反応するのです。これは、最初に設定した `channel` の受信側で入力を待つだけで実現出来ます。

```rust
        match rx.recv()? {
            Event::Input(event) => match event.code {
                KeyCode::Char('q') => {
                    disable_raw_mode()?;
                    terminal.show_cursor()?;
                    break;
                }
                KeyCode::Char('h') => active_menu_item = MenuItem::Home,
                KeyCode::Char('p') => active_menu_item = MenuItem::Pets,
                _ => {}
            },
            Event::Tick => {}
        }
} // end of render loop
```

イベントが来たら、入力されたキーコードにマッチします。ユーザーが `q` を押したら、アプリを終了させたい。つまり、クリーンアップを行い、取得したのと同じ状態で端末を返却する。実際のアプリケーションでは、致命的なエラーが発生した場合にも、このクリーンアップが行われるはずです。そうしないと、ターミナルが `raw` モードのままになってしまい、見た目がおかしくなってしまいます。

`raw` モードを無効にし、カーソルを再び表示することで、ユーザーはアプリを起動する前と同様に通常の端末状態になるはずです。

`h` があればアクティブメニューを `Home` に、 `p` があれば `Pets` に設定します。これが必要なルーティングロジックの全てです。

また、 `terminal.draw` クロージャの中で、このルーティングのためのロジックを追加する必要があります。

```rust
            match active_menu_item {
                MenuItem::Home => rect.render_widget(render_home(), chunks[1]),
                MenuItem::Pets => {
                  ...
                }
            }
...

fn render_home<'a>() -> Paragraph<'a> {
    let home = Paragraph::new(vec![
        Spans::from(vec![Span::raw("")]),
        Spans::from(vec![Span::raw("Welcome")]),
        Spans::from(vec![Span::raw("")]),
        Spans::from(vec![Span::raw("to")]),
        Spans::from(vec![Span::raw("")]),
        Spans::from(vec![Span::styled(
            "pet-CLI",
            Style::default().fg(Color::LightBlue),
        )]),
        Spans::from(vec![Span::raw("")]),
        Spans::from(vec![Span::raw("Press 'p' to access pets, 'a' to add random new pets and 'd' to delete the currently selected pet.")]),
    ])
    .alignment(Alignment::Center)
    .block(
        Block::default()
            .borders(Borders::ALL)
            .style(Style::default().fg(Color::White))
            .title("Home")
            .border_type(BorderType::Plain),
    );
    home
}
```

`active_menu_item` でマッチングし、それが `Home` であれば、ユーザーの現在地とアプリの操作方法を伝える基本的なウェルカムメッセージをレンダリングするだけです。（スタート画面）

入力処理に関しては、今のところこれで全てです。これを実行すれば、すでにタブをいじることができます。しかし、ペット管理アプリケーションの本題である、ペットの扱いが抜けています。

# TUI でステートフルウェジットを作成

最初のステップは、JSON ファイルからペットを読み込んで、左側にペットの名前のリストを表示し、右側に選択したペット（デフォルトは 1 番目）の詳細情報を表示することです。

ここで少し免責事項として、このチュートリアルの範囲が限られているため、このアプリケーションはエラーを適切に処理しません。ファイルの読み込みに失敗した場合、アプリは役立つエラーを表示する代わりにクラッシュします。基本的には、`=` エラー処理は他の UI アプリケーションと同じように動作します。エラーで分岐し、例えば、問題を解決するためのヒントや行動を呼びかけるような、役立つエラー・パラグラフをユーザーに表示することができます。

それは置いといて、 `active_menu_item` のマッチングの欠落部分をみてみます。

```rust
            match active_menu_item {
                MenuItem::Home => rect.render_widget(render_home(), chunks[1]),
                MenuItem::Pets => {
                    let pets_chunks = Layout::default()
                        .direction(Direction::Horizontal)
                        .constraints(
                            [Constraint::Percentage(20), Constraint::Percentage(80)].as_ref(),
                        )
                        .split(chunks[1]);
                    let (left, right) = render_pets(&pet_list_state);
                    rect.render_stateful_widget(left, pets_chunks[0], &mut pet_list_state);
                    rect.render_widget(right, pets_chunks[1]);
                }
            }
```

`Pets` のページにいる場合は、新しいレイアウトを作成します。今回は、リストビューと詳細ビューという 2 つの要素を隣り合わせに表示するため、横型のレイアウトにします。この場合、リストビューが画面の約 20%を占め、残りは詳細テーブルのために残しておきます。

`.split` では、`rect` サイズの代わりに `chunks[1]` を使用していることに注意してください。これは、 `chunks[1]` （中央の `Content` レイアウトチャンク）の領域を記述する短形を、2 つの水平レビューに分割することを意味しています。次に、 `render_pets` を呼び出して、レンダリングする左右の部分を返し、それらに対応する `pets_chunks` に単純にレンダリングしています。

`render_stateful_widget` は何をするのでしょうか。

TUI は、完全で素晴らしいフレームワークであるため、ステートフルなウィジェットを作成することができます。リスト・ウェジットは、ステートフルなウィジェットの 1 つです。これを動作させるには、レンダリングループの前に、 `ListState` を最初に作成する必要があります。

```rust
    let mut pet_list_state = ListState::default();
    pet_list_state.select(Some(0));

loop {
...
```

ペットリストの状態を初期化し、デフォルトで最初のアイテムを選択します。この `pet_list_state` は、 `render_pets` 関数に渡すものでもあります。この関数はすべてのロジックを実行します。

- データベースファイルからペットを取得する
- ペットをリスト項目に変換する
- ペットのリストを作成する
- 現在選択されているペットを検索する
- 選択したペットのデータをテーブルに表示する、ペットのデータでテーブルをレンダリングする
- 両方のウィジェットを返す

かなり多いので、コードは少し長くなります。

```rust
fn read_db() -> Result<Vec<Pet>, Error> {
    let db_content = fs::read_to_string(DB_PATH)?;
    let parsed: Vec<Pet> = serde_json::from_str(&db_content)?;
    Ok(parsed)
}

fn render_pets<'a>(pet_list_state: &ListState) -> (List<'a>, Table<'a>) {
    let pets = Block::default()
        .borders(Borders::ALL)
        .style(Style::default().fg(Color::White))
        .title("Pets")
        .border_type(BorderType::Plain);

    let pet_list = read_db().expect("can fetch pet list");
    let items: Vec<_> = pet_list
        .iter()
        .map(|pet| {
            ListItem::new(Spans::from(vec![Span::styled(
                pet.name.clone(),
                Style::default(),
            )]))
        })
        .collect();

    let selected_pet = pet_list
        .get(
            pet_list_state
                .selected()
                .expect("there is always a selected pet"),
        )
        .expect("exists")
        .clone();

    let list = List::new(items).block(pets).highlight_style(
        Style::default()
            .bg(Color::Yellow)
            .fg(Color::Black)
            .add_modifier(Modifier::BOLD),
    );

    let pet_detail = Table::new(vec![Row::new(vec![
        Cell::from(Span::raw(selected_pet.id.to_string())),
        Cell::from(Span::raw(selected_pet.name)),
        Cell::from(Span::raw(selected_pet.category)),
        Cell::from(Span::raw(selected_pet.age.to_string())),
        Cell::from(Span::raw(selected_pet.created_at.to_string())),
    ])])
    .header(Row::new(vec![
        Cell::from(Span::styled(
            "ID",
            Style::default().add_modifier(Modifier::BOLD),
        )),
        Cell::from(Span::styled(
            "Name",
            Style::default().add_modifier(Modifier::BOLD),
        )),
        Cell::from(Span::styled(
            "Category",
            Style::default().add_modifier(Modifier::BOLD),
        )),
        Cell::from(Span::styled(
            "Age",
            Style::default().add_modifier(Modifier::BOLD),
        )),
        Cell::from(Span::styled(
            "Created At",
            Style::default().add_modifier(Modifier::BOLD),
        )),
    ]))
    .block(
        Block::default()
            .borders(Borders::ALL)
            .style(Style::default().fg(Color::White))
            .title("Detail")
            .border_type(BorderType::Plain),
    )
    .widths(&[
        Constraint::Percentage(5),
        Constraint::Percentage(20),
        Constraint::Percentage(20),
        Constraint::Percentage(5),
        Constraint::Percentage(20),
    ]);

    (list, pet_detail)
}
```

まず、`read_db` 関数を定義します。この関数は単に JSON ファイルを読み込んでそれを `Pets` の `Vec` にパースします。

そして、 `List` と `Table` （どちらも TUI ウィジェット）のタプルを返す `render_pets` の中で、まずリストビューのための `pets` ブロックを囲むように定義します。

ペットのデータを取得した後、ペットの名前を `ListItems` に変換しています。次に、 `pet_list_state` に基づいて、リスト内で選択されたペットを見つけようとします。これが失敗したり、ペットがいなかったりすると、このシンプルなバージョンではアプリがクラッシュします。なぜなら、前述のように意味のあるエラーハンドリングがないからです。

選択されたペットを取得したら、リストウィジェットを作成してリストアイテムを表示し、ハイライトスタイルを定義して、現在どのペットが選択されているかを確認出来るようにします。

最後に、 `pet_details` テーブルを作成し、ハードコードされたヘッダーをペット構造体の列名に設定します。また、各ペットのデータを文字列に変換した `Rows` のリストを定義します。

このテーブルを、タイトル「Details」とボーダーを持つ基本ブロックにレンダリングし、5 つの絡むの相対幅をパーセントで定義します。そのため、このテーブルはリサイズ時にもレスポンシブに動作します。

これがペットの全レンダリングロジックです。あとは、ランダムに新しいペットを追加する機能と、選択したペットを削除する機能を追加して、あぷりにもう少しインタラクティブ性を持たせるだけです。

まず、ペットの追加と削除のための 2 つのヘルパー関数を追加します。

```rust
fn add_random_pet_to_db() -> Result<Vec<Pet>, Error> {
    let mut rng = rand::thread_rng();
    let db_content = fs::read_to_string(DB_PATH)?;
    let mut parsed: Vec<Pet> = serde_json::from_str(&db_content)?;
    let catsdogs = match rng.gen_range(0, 1) {
        0 => "cats",
        _ => "dogs",
    };

    let random_pet = Pet {
        id: rng.gen_range(0, 9999999),
        name: rng.sample_iter(Alphanumeric).take(10).collect(),
        category: catsdogs.to_owned(),
        age: rng.gen_range(1, 15),
        created_at: Utc::now(),
    };

    parsed.push(random_pet);
    fs::write(DB_PATH, &serde_json::to_vec(&parsed)?)?;
    Ok(parsed)
}

fn remove_pet_at_index(pet_list_state: &mut ListState) -> Result<(), Error> {
    if let Some(selected) = pet_list_state.selected() {
        let db_content = fs::read_to_string(DB_PATH)?;
        let mut parsed: Vec<Pet> = serde_json::from_str(&db_content)?;
        parsed.remove(selected);
        fs::write(DB_PATH, &serde_json::to_vec(&parsed)?)?;
        pet_list_state.select(Some(selected - 1));
    }
    Ok(())
}
```

どちらの場合も、ペットリストを読み込んで、操作してファイルに書き戻します。`remove_pet_at_index` の場合、 `pet_list_state` をデクリメントする必要があるので、リスト内の最後のペットにいる場合は、自動的に前のペットにジャンプします。

最後に、ペットリスト内で上下に移動するための入力ハンドラと、ペットの追加と削除に使用する `a` および `d` を追加する必要があります。

```rust
		KeyCode::Char('a') => {
				add_random_pet_to_db().expect("can add new random pet");
		}
		KeyCode::Char('d') => {
				remove_pet_at_index(&mut pet_list_state).expect("can remove pet");
		}
		KeyCode::Down => {
				if let Some(selected) = pet_list_state.selected() {
						let amount_pets = read_db().expect("can fetch pet list").len();
						if selected >= amount_pets - 1 {
								pet_list_state.select(Some(0));
						} else {
								pet_list_state.select(Some(selected + 1));
						}
				}
		}
		KeyCode::Up => {
				if let Some(selected) = pet_list_state.selected() {
						let amount_pets = read_db().expect("can fetch pet list").len();
						if selected > 0 {
								pet_list_state.select(Some(selected - 1));
						} else {
								pet_list_state.select(Some(amount_pets - 1));
						}
				}
		}
```

上と下の場合、リストがアンダーフローやオーバーフローしないようにする必要があります。この単純な例では、これらのキーが押される度にペットリストを読み取るだけなので、非常に非効率ですが、十分に単純です。実際のアプリでは、現在のペットの数を共有メモリのどこかに保持することも出来ますが、このチュートリアルでは、この単純な方法で十分です。

いずれにしても、 `pet_list_state` の値を単純にインクリメント（Up の場合はデクリメント）することで、ユーザーは `Up` と `Down` を使ってペットのリストをスクロールできるようになります。 `cargo run` を使用してアプリケーションを起動すると、さまざまなキーバインドをテストでき、`a` を使用してランダムにペットを追加し、移動して `d` を使用して再び削除することができます。
