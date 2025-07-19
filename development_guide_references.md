読者になる
little hands' lab
ドメイン駆動設計、アジャイルプラクティスを実践し、解説しています。
トップ > ドメイン駆動設計(DDD) > DDD基礎解説：エンティティ、値オブジェクトってなんなんだ
 2024-03-17
DDD基礎解説：エンティティ、値オブジェクトってなんなんだ
ドメイン駆動設計(DDD)
はじめに

DDDの実装パターンとして、エンティティと値オブジェクトというものがあります。
ドメイン駆動一般に複雑な抽象論が多い中で、コードに近く一番イメージがつきやすいコード事例として出てくるため、ここだけは何となくわかるぞ！という方もいらっしゃるのではないでしょうか。
今日はこちらの概要とそれぞれの使い道について書きたいと思います。

先にざっくりイメージ図をお伝えすると、こういう図を使って解説します。

何の目的で作るのか？

ドメイン駆動設計は何を解決しようとしているのか こちらの記事で、ドメイン駆動設計のアプローチは以下の2ステップがあるということを書きました。

ドメインの問題を解決するための抽象的なモデルを作る.
モデルをソフトウェア(コード)に落とし込む

※ ドメイン＝ソフトウェアを適用して問題解決しようとする領域

DDDでは、このStep2の モデルをコードで表現するためのパターン として、以下の4つを定義しています。

エンティティ
値オブジェクト
ドメインイベント
ドメインサービス

DDD Refference より一部抜粋 "Express Model With"と書かれている4つ


このうち、 ドメインの知識を「モノ」として表現する のがエンティティと値オブジェクトの2つです。使用頻度が高く理解しやすいため、初学者はまずこの2つから学ぶとよいでしょう。本記事では、エンティティと値オブジェクトについて詳しく解説します。

ドメインサービスとドメインイベント

今回の趣旨からは一瞬それますが、残りの二つを軽くだけ説明します。

ドメインサービスは、ドメインの知識を「手続きやプロセス」として表現するものです。 「モノ」「コト」として表現すると無理があるものの表現に使います。例えば、集合に対する操作などです。

例として、「予約」というオブジェクト自体があったとします。
"指定された特定の時間帯に予約の空きがあるか？"と尋ねられたとき、その知識を予約オブジェクト自身が答えられる、と表現するのは無理があります。自分自身の予約時間を知っていても、それ以外のオブジェクトの状況については情報として持っていないからです。
こういう場合には、「時間の重複をなく予約を生成するという手続き」をドメインサービスというものを使用して表現します。

ただ、ドメインサービスには注意が2点あります。

まず、ドメインサービスはつい手続き的な書き方になってしまうということです。極力エンティティと値オブジェクトで表現できないかを検討して、どうしてもできない時のみ使うようにします。
もう一つは、使うとしても「サービス」という名称はお勧めしないということです。「サービス」と名付けられたクラスは責務が曖昧になり、肥大化していきがちです。「それは何をするクラスなのか？」と責務を明確にし「XxxRegisterer」「XxxApprover」などと責務を表現した名前にすることをお勧めします。




ドメインイベントは、ドメインの知識を「コト」として表現するものです。「予約」をエンティティ/値オブジェクトとするなら、「予約が行われた」と言った形で表現されるのがドメインイベントです。

このモデルを別のドメインサービスなどで拾って別の処理を行うといった使い方をします。
ただ、残りの3つに比べるとかなり応用編といったところなので、初学者は一旦は置いておいて後からの理解で大丈夫です。




それでは、ドメインイベントとドメインサービスについては軽く抑えたところで、エンティティと値オブジェクトの紹介に移りたいと思います。

エンティティと値オブジェクトの違い

同じ「ドメイン知識を『モノ』として表現するもの」であるのに、なぜ2つあるのでしょうか？その違いさえ理解できれば、使い分けることができるようになるでしょう。

	エンティティ	値オブジェクト
同一性判定	識別子が同一であれば同一	保持する属性が全て同一であれば同一
可変性	可変でもよい	必ず不変

定義としては、同一性の判定方法がどちらか、というものになりますが、可変性も性質として重要なので併記しています。

エンティティ
エンティティの同一性判定と可変性

社員というエンティティについて考えます。

例えば、山田さんという社員は、ある会社においては社員番号123という識別子で同一判定されます。山田さんは部署が変わろうが、所持金が変わろうが、体重が変わろうが同じ「山田さん」であり、別人にはなりませんね。

一方、新しく名前が同じ山田さんという社員が入ってきて社員番号456が割り振られたとします。この人は部署、所持金、体重が仮に全部同じだったとしても、123の山田さんとは別の人物です。これがエンティティの同一性の考え方となります。

値オブジェクト
値オブジェクトの同一性判定

一方、お金について考えます。

2つの10円玉が並んでいて、これを「同じ」と判断したいでしょうか？

それは 文脈、モデリングの目的による ものとなります。
モデリング時の興味が、そのものが表す金銭的価値にしか興味がないとすると、2つの10円玉は同じと考えてよいことになります。
例えば、山田さんが1つ目の10円玉を持っている状況と2つ目の10円玉を持っている状況では、等しく山田さんのの所持金は10円と考えたいのではないでしょうか。この場合、二つをの10円を区別する必要はありません。このような場合、10円は値オブジェクトとしてモデリングする方が適切です。

もしこれが造幣局やコインコレクターだとするならば事情は異なるかもしれません。それぞれの10円玉を区別して扱いたい可能性はあり得ます。これが先ほど書いた「文脈、モデリングの目的による」という意味です。
その場合はお金をエンティティとしてモデリングするということも検討することになります。

値オブジェクトの不変性

山田さんが所持金として10円を持っていたところ、100円に増えたとします。この時に10円玉の数字の10に取り消し線を書いて、100と書き直すでしょうか？いや、そんなことはしませんね。

実際には、10円玉(という値オブジェクト)を、100円玉(という値オブジェクト)と交換するでしょう。10円玉は製造された時に保持する金銭的価値は確定しており、あとからいかなることがあっても変わることはない。これが値オブジェクトが不変であるということです。

エンティティと値オブジェクトの関係

先ほどの社員とお金のように、エンティティが自身の属性として値オブジェクトを保持するという関係になるのが基本となります。
エンティティの属性は可変ですが、値オブジェクトとして持つ属性の値が変わる場合は、値オブジェクト自体が示す値を変えるのではなく、新しい値を持つ値オブジェクトを生成し、前の値オブジェクトを置き換えるという使い方します。10円玉を100円玉で置き換えるというのは、まさにこの事例です。

このような設計をする意図
エンティティと値オブジェクトを区別する利点

なぜこのような区別をするのでしょうか？ これは端的に実装上の利点があるからです。 これは私の実体験からの解釈ですが、以下の2つが挙げられると考えています。

① ドメインの知識をそのまま表現しやすくなる
② 可変性を最小限に抑えられる

この2つを実現することで、保守性を向上できます。

先ほどの例で、値をオブジェクトをエンティティとして表現すると、「10円玉自体が硬貨IDを保持し、数値部分を書き換えられる」という実装になってしまいます。これは直感的に現実世界(ドメイン)の知識の表現からは乖離してしまいますね。今回の硬貨のようなものは、「識別子ではなく属性で判断する値」とすると表現しやすい、というパターンだと理解してもらえると良いと思います。

また、可変性を最小限に、というのは、最近は多くのプログラミング言語にも取り入れられている方針であり、一般的なプログラミング言語のプラクティスであるため、ここではあまり掘り下げません。

プリミティブ型ではなく値オブジェクトとして設計する利点

値オブジェクトについて、エンティティとの対比として利点を書きましたが、プリミティブ型と対比すると、きちんと値自体に振る舞いを持たせることで凝集度を上げることが可能になります。

例えば、社員というエンティティがメールアドレスを持つ場合、メールアドレスの書式チェックを社員エンティティが持つよりも、メールアドレスという値オブジェクトの生成時のロジック(コンストラクタ等)に書式チェックを持たせる方が、オブジェクトの持つ値と振る舞いの関連度が近いので、凝集度が高いということができます。

もっと詳しく知りたい方は

DDDを学び、実践したい方のために、2冊の書籍を執筆しました。

①基礎的な概念や考え方を学びたい方は

little-hands.booth.pm

初めてDDDを学ぶ方、もしくは実際に着手して難しさにぶつかっている方向けの書籍です。

DDDの目的やモデリングの考え方から始まり、具体的な実装パターンまで幅広く解説しています。特に「第6章 ドメイン層の実装」では、本記事で触れなかったドメインサービス、ドメインイベント、リポジトリ、ファクトリーなどの重要な実装パターンについて解説しています。
基礎的な概念や用語を理解する際にお役に立ちます。

②実際のコードを見ながら実践したい方は

little-hands.booth.pm

実践にあたって頻出の疑問に対して、トピックごとに詳しく解説した書籍です。

重要トピック「モデリング」「集約」「テスト」について詳細に解説し、その他のトピックでは頻出の質問への回答と具体的なサンプルコードをふんだんに盛り込みました。現場で実践して、困っていることがある方はぜひこちらもご覧ください。

この本の「第3章 エンティティ/値オブジェクト」では、「データモデルやOR マッパークラスとの違い」「エンティティの生成方法」「DB のオートインクリメントの値をID に使用してよい？」と言った具体的な疑問に詳細なコード付きで解説しています。

「モデリング/実装ガイド」で基礎概念を理解した後は、こちらの書籍を読んでいただけると実際のコードを見ながら実践するのに役立ちます。

現場での導入で困ったら

DDDを導入しようとすると結構試行錯誤に時間がかかります。
現場で導入してすぐに効果を発揮したい！！という方向けに、基礎解説とライブモデリング/コーディングを行う勉強会の開催や、設計相談を受付けております。事例紹介もあるのでご関心あれば覗いてみてください。
開催形式は柔軟に対応できるのでお気軽にご相談ください。

little-hand-s.notion.site

Twitterでも、DDDに関して発信したり、「質問箱」というサービスを通じて質問を受け付けています。こちらもよろしければフォローしてください。

https://twitter.com/little_hand_s

また、YouTubeで10分でわかるDDD動画シリーズをアップしています。概要を動画で理解したい方はこちらもどうぞ。チャンネル登録すると新しい動画の通知を受け取ることができます。

little_hands 1年前 読者になる

関連記事
2022-06-01
簡単にできるDDDのモデリング - ドメイン駆動設計
DDDではよく「モデリングが重要だ！」と言われますが、どのよう…
2022-01-28
設計/コードレビューで"常に"心がけるポイント
株式会社ログラスの松岡(@little_hand_s)です。 little-hands.h…
2021-12-13
DDDのエンティティはイミュータブルな実装にしてもいいの？(サンプルコード有り)[ドメイン駆動設計 …
本記事はドメイン駆動設計（ＤＤＤ） Advent Calendar 2021の13…
2019-08-31
「DDDのモデリングとは何なのか、 そしてどうコードに落とすのか」資料 / Q&A
Mix Leap Study 特別編 - レガシーをぶっつぶせ。現場でDDD！ …
2018-12-17
非エンジニアの方に「DDDって何なの？」と聞かれたときの説明[ドメイン駆動設計]
この記事はドメイン駆動設計 #2 Advent Calendar 2018の16日目…

nanndemoiikara

s/識別子供/識別子/

6年前 

nanndemoiikara

s/モデリングすということ/モデリングするということ/

6年前 

コメントを書く
« 人気&オススメ記事 / ブログ概要
DDDにおける値オブジェクトの位置付け(モ… »
Profile
little_hands はてなブログPro
最終更新: 2年前

ドメイン駆動設計について、実践のきっかけにできるような記事を書いていきたいと思います。

読者になる
629
このブログについて
Category
ドメイン駆動設計(DDD) (32)
コミュニケーション (2)
開発手法 (1)
仕事術 (4)
プログラミング一般 (1)
python (1)
Hibernate (1)
リファクタ (1)
設計 (2)
ロジカルシンキング (1)
Excel (1)
最新記事
人気&オススメ記事 / ブログ概要
DDD基礎解説：エンティティ、値オブジェクトってなんなんだ
DDDにおける値オブジェクトの位置付け(モデルとコード事例あり)[ドメイン駆動設計]
アジャイル迷子のための「アジャイルの本質」。あとDDDとのつながり
簡単にできるDDDのモデリング - ドメイン駆動設計
Related Entry
人気&オススメ記事 / ブログ概要
DDDにおける値オブジェクトの位置付け(モデルとコード事例あり)[ドメイン駆動設計]
DDDにおけるドメイン層オブジェクト設計の基本方針[ドメイン駆動設計]
Hot Entries
DDD基礎解説：エンティティ、値オブジェクトってなんなんだ
新卒にも伝わるドメイン駆動設計のアーキテクチャ説明(オニオンアーキテクチャ)[DDD]
境界づけられたコンテキスト 概念編 - ドメイン駆動設計用語解説 [DDD]
簡単にできるDDDのモデリング - ドメイン駆動設計
CQRS実践入門 [ドメイン駆動設計]
Archive
▶ 9999 (1)
▼ 2024 (1)
2024 / 3 (1)
▶ 2022 (5)
▶ 2021 (4)
▶ 2020 (3)
▶ 2019 (11)
▶ 2018 (10)
▶ 2017 (20)
▶ 2014 (4)
検索
はてなブログをはじめよう！

little_handsさんは、はてなブログを使っています。あなたもはてなブログをはじめてみませんか？

はてなブログをはじめる（無料）
はてなブログとは
 little hands' lab

Powered by Hatena Blog | ブログを報告する 読者になる
little hands' lab
ドメイン駆動設計、アジャイルプラクティスを実践し、解説しています。
トップ > ドメイン駆動設計(DDD) > 新卒にも伝わるドメイン駆動設計のアーキテクチャ説明(オニオンアーキテクチャ)[DDD]
 2018-12-12
新卒にも伝わるドメイン駆動設計のアーキテクチャ説明(オニオンアーキテクチャ)[DDD]
ドメイン駆動設計(DDD)

ドメイン駆動設計で実装を始めるのに一番とっつきやすいアーキテクチャは何か
ドメイン駆動 + オニオンアーキテクチャ概略

以前こちらの記事でアプリケーションアーキテクチャについて書きました。
こちらの記事では比較的ネタ元に忠実な解説をしたのですが、実際これに基づいて人にレイヤの説明をした際、依存性の逆転部分や円形で表現する部分がなかなか伝わりにくいことがありました。

そんな中で、所属プロダクトで新卒含めて大規模なリニューアル案件でDDDを採用することになり、新卒にも伝わるように説明をする必要性が生じました。
結果、新卒にも伝わり、運用が割と回る説明が見つかったのでご紹介したいと思います。

アプリケーションアーキテクチャ全体図

とにかく、何か説明する際はこの図を常に傍に置き、一方通行の依存性を徹底したい、という話をしています。
何かについて議論をする際は、 「それはどの層の責務なの？」 という話をして、この図を元に「ではここに書かないといけないね」 という話をしています。

特に、ドメインモデル図にドメインの振る舞いを書いたあと 「この知識はUseCaseではなくDomain層に書きたいから、UseCaseに書いたらリファクタしよう」 ということは繰り返し伝えています。(UseCaseレベルのテストがあればあとはなんとでもリファクタできます)

徐々に伝わってくると、「これはドメインの知識なのでドメイン層に・・・」という会話が徐々に生まれてきてくれました。

(オニオンアーキテクチャとか、細かい話をするのは一旦やめました。笑)

以前の図との変更点

参考までに書いておくと、

・ApplicationServiceという呼称をUseCaseに変更した ・Infra層からDomain層への依存を実装の矢印で表現した
・Domain層をModelとServiceに分けるのはやめて一つにした
・登場要素をマッピングした

あたりが変更点です。

ApplicationServiceという呼称をUseCaseに変更した点については、「これはApplicationの関心ごとだよね」と言ってもApplicationが多義語すぎて人によって解釈がブレてしまったのに比べて、「ドメインの知識はDomain層に、ユースケースの実現はUseCase層に」という説明の方が伝わりやすかったためです。なお、ApplicationServiceは元々オニオンアーキテクチャの呼称、UseCaseというのはCleanArchitectureの呼称なので、ここだけはCleanArchitectureのエッセンスを取り入れた形になります。

Infra層からDomain層への依存を実装の矢印で表現した点に関しては、やはり以下の図では依存の形がイメージしにくかったためです。

これで何を指すのかがだいぶ一目で伝わりやすくなりました。

後からの補足として、依存関係逆転の説明

依存性逆転の法則については、実装を少ししてイメージが湧いた段階になってから、こんな説明をしました。

デメリットの

・DB変更のたびにEntityやUseCaseを変更する可能性がある
・DBの都合に引きずられた実装になる

あたりについて、新しいアーキテクチャの実装イメージが湧き、そうではないアーキテクチャでの苦しい経験が思い出されるようになると、「確かに〜」と納得してもらえるようになりました。

使い道

これを見てすぐにDDDの実装ができるようになる訳ではありませんが、議論のベースに可視化された一枚絵があるのはとても便利に感じます。そう言った使い身としてでもお役に立てると幸いです。

もっと詳しく知りたい方は

little-hands.booth.pm

初めてDDDを学ぶ方、もしくは実際に着手して難しさにぶつかっている方向けの書籍を出しました。
迷子になりがちな「DDDの目的」や「モデル」の解説からはじめ、 具体的なモデリングを行い実装まで落とす事例を元に、DDDの魅力や効果を体感することを目指します。

この本の「第5章 アーキテクチャ」では、本ブログ記事の内容に加えて、他のアーキテクチャと比較しながらさらに詳しく解説しています。 よろしければお求めください。

また、実践にあたって頻出の疑問に対してトピックごとに詳しく解説した書籍があります。

重要トピック「モデリング」「集約」「テスト」について詳細に解説し、その他のトピックでは頻出の質問への回答と具体的なサンプルコードをふんだんに盛り込みました。現場で実践して、困っていることがある方はぜひこちらもご覧ください。

little-hands.booth.pm

現場での導入で困ったら

DDDを導入しようとすると結構試行錯誤に時間がかかります。
現場で導入してすぐに効果を発揮したい！！という方向けに、基礎解説とライブモデリング/コーディングを行う勉強会の開催や、設計相談を受付ております。
事例紹介もあるのでご関心あれば覗いてみてください。開催形式は柔軟に対応できるのでお気軽にご相談ください。

little-hand-s.notion.site

Twitterでも、DDDに関して発信したり、「質問箱」というサービスを通じて質問を受け付けています。こちらもよろしければフォローしてください

https://twitter.com/little_hand_s

また、YouTubeで10分でわかるDDD動画シリーズをアップしています。概要を動画で理解したい方はこちらもどうぞ。チャンネル登録すると新しい動画の通知を受け取ることができます。

little_hands 6年前 読者になる

関連記事
9999-12-31
人気&オススメ記事 / ブログ概要
当ブログについて 主にドメイン駆動設計(DDD)関連の情報を発信…
2022-01-28
設計/コードレビューで"常に"心がけるポイント
株式会社ログラスの松岡(@little_hand_s)です。 little-hands.h…
2020-12-22
ドメイン駆動設計を導入するために転職して最初の3ヶ月でやったこと[DDD]
この記事は ドメイン駆動設計 Advent Calendarの記事です。 今…
2019-08-31
「DDDのモデリングとは何なのか、 そしてどうコードに落とすのか」資料 / Q&A
Mix Leap Study 特別編 - レガシーをぶっつぶせ。現場でDDD！ …
2018-04-04
オニオンアーキテクチャにておいて、ドメイン層とアプリケーション層の責務はどう違うのか[DDD]
ドメイン駆動設計で実装を始めるのに一番とっつきやすいアーキ…
コメントを書く
« 非エンジニアの方に「DDDって何なの？」と…
ドメイン駆動設計は何を解決しようとして… »
Profile
little_hands はてなブログPro
最終更新: 2年前

ドメイン駆動設計について、実践のきっかけにできるような記事を書いていきたいと思います。

読者になる
629
このブログについて
Category
ドメイン駆動設計(DDD) (32)
コミュニケーション (2)
開発手法 (1)
仕事術 (4)
プログラミング一般 (1)
python (1)
Hibernate (1)
リファクタ (1)
設計 (2)
ロジカルシンキング (1)
Excel (1)
最新記事
人気&オススメ記事 / ブログ概要
DDD基礎解説：エンティティ、値オブジェクトってなんなんだ
DDDにおける値オブジェクトの位置付け(モデルとコード事例あり)[ドメイン駆動設計]
アジャイル迷子のための「アジャイルの本質」。あとDDDとのつながり
簡単にできるDDDのモデリング - ドメイン駆動設計
Related Entry
人気&オススメ記事 / ブログ概要
DDD基礎解説：エンティティ、値オブジェクトってなんなんだ
DDDにおける値オブジェクトの位置付け(モデルとコード事例あり)[ドメイン駆動設計]
Hot Entries
DDD基礎解説：エンティティ、値オブジェクトってなんなんだ
新卒にも伝わるドメイン駆動設計のアーキテクチャ説明(オニオンアーキテクチャ)[DDD]
境界づけられたコンテキスト 概念編 - ドメイン駆動設計用語解説 [DDD]
簡単にできるDDDのモデリング - ドメイン駆動設計
CQRS実践入門 [ドメイン駆動設計]
Archive
▶ 9999 (1)
▶ 2024 (1)
▶ 2022 (5)
▶ 2021 (4)
▶ 2020 (3)
▶ 2019 (11)
▼ 2018 (10)
2018 / 12 (2)
2018 / 10 (1)
2018 / 9 (1)
2018 / 8 (1)
2018 / 5 (1)
2018 / 4 (2)
2018 / 1 (2)
▶ 2017 (20)
▶ 2014 (4)
検索
はてなブログをはじめよう！

little_handsさんは、はてなブログを使っています。あなたもはてなブログをはじめてみませんか？

はてなブログをはじめる（無料）
はてなブログとは
 little hands' lab

Powered by Hatena Blog | ブログを報告する Organizing Layers Using Hexagonal Architecture, DDD, and Spring

Last updated: May 11, 2024

Written by:
Łukasz Ryś
Reviewed by:
Josh Cummings
ArchitectureSpring+
DDD
1. Overview

In this tutorial, we'll implement a Spring application using DDD. Additionally, we'll organize layers with the help of Hexagonal Architecture.

With this approach, we can easily exchange the different layers of the application.

2. Hexagonal Architecture

Hexagonal architecture is a model of designing software applications around domain logic to isolate it from external factors.

The domain logic is specified in a business core, which we'll call the inside part, with the rest being outside parts. Access to domain logic from the outside is available through ports and adapters. 

3. Principles

First, we should define principles to divide our code. As explained briefly already, hexagonal architecture defines the inside and outside part.

What we'll do here is divide our application into three layers: application (outside), domain (inside), and infrastructure (outside):

Through the application layer, the user or any other program interacts with the application. This area should contain things like user interfaces, RESTful controllers, and JSON serialization libraries. It includes anything that exposes entry to our application, and orchestrates the execution of domain logic.

In the domain layer, we keep the code that touches and implements business logic. This is the core of our application. This layer should be isolated from both the application part and infrastructure part. In addition, it should also contain interfaces that define the API to communicate with external parts, like the database, which the domain interacts with.

Finally, the infrastructure layer is the part that contains anything that the application needs to work, such as database configuration or Spring configuration. It also implements infrastructure-dependent interfaces from the domain layer.

4. Domain Layer

Let's begin by implementing our core layer, which is the domain layer.

First, we should create the Order class:

public class Order {
    private UUID id;
    private OrderStatus status;
    private List<OrderItem> orderItems;
    private BigDecimal price;

    public Order(UUID id, Product product) {
        this.id = id;
        this.orderItems = new ArrayList<>(Arrays.astList(new OrderItem(product)));
        this.status = OrderStatus.CREATED;
        this.price = product.getPrice();
    }

    public void complete() {
        validateState();
        this.status = OrderStatus.COMPLETED;
    }

    public void addOrder(Product product) {
        validateState();
        validateProduct(product);
        orderItems.add(new OrderItem(product));
        price = price.add(product.getPrice());
    }

    public void removeOrder(UUID id) {
        validateState();
        final OrderItem orderItem = getOrderItem(id);
        orderItems.remove(orderItem);

        price = price.subtract(orderItem.getPrice());
    }

    // getters
}
Copy

This is our aggregate root. Anything related to our business logic will go through this class. Additionally, Order is responsible for keeping itself in the correct state:

The order can only be created with the given ID and based on one Product; the constructor itself also inits the order with CREATED status.
Once the order is completed, changing OrderItems is impossible.
It's impossible to change the Order from outside the domain object, like with a setter.

Furthermore, the Order class is also responsible for creating its OrderItem.

So let's create the OrderItem class:

public class OrderItem {
    private UUID productId;
    private BigDecimal price;

    public OrderItem(Product product) {
        this.productId = product.getId();
        this.price = product.getPrice();
    }

    // getters
}
Copy

As we can see, OrderItem is created based on a Product. It keeps the reference to it and stores the current price of the Product.

Next, we'll create a repository interface (a port in Hexagonal Architecture). The implementation of the interface will be in the infrastructure layer:

public interface OrderRepository {
    Optional<Order> findById(UUID id);

    void save(Order order);
}
Copy

Finally, we should make sure that the Order will always be saved after each action. To do that, we'll define a Domain Service, which usually contains logic that can't be a part of our root:

public class DomainOrderService implements OrderService {

    private final OrderRepository orderRepository;

    public DomainOrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public UUID createOrder(Product product) {
        Order order = new Order(UUID.randomUUID(), product);
        orderRepository.save(order);

        return order.getId();
    }

    @Override
    public void addProduct(UUID id, Product product) {
        Order order = getOrder(id);
        order.addOrder(product);

        orderRepository.save(order);
    }

    @Override
    public void completeOrder(UUID id) {
        Order order = getOrder(id);
        order.complete();

        orderRepository.save(order);
    }

    @Override
    public void deleteProduct(UUID id, UUID productId) {
        Order order = getOrder(id);
        order.removeOrder(productId);

        orderRepository.save(order);
    }

    private Order getOrder(UUID id) {
        return orderRepository
          .findById(id)
          .orElseThrow(RuntimeException::new);
    }
}
Copy

In a hexagonal architecture, this service is an adapter that implements the port. We won't register it as a Spring bean because, from a domain perspective, this is in the inside part, and Spring configuration is on the outside. We'll manually wire it with Spring in the infrastructure layer a bit later.

Because the domain layer is completely decoupled from the application and infrastructure layers, we can also test it independently:

class DomainOrderServiceUnitTest {

    private OrderRepository orderRepository;
    private DomainOrderService tested;
    @BeforeEach
    void setUp() {
        orderRepository = mock(OrderRepository.class);
        tested = new DomainOrderService(orderRepository);
    }

    @Test
    void shouldCreateOrder_thenSaveIt() {
        final Product product = new Product(UUID.randomUUID(), BigDecimal.TEN, "productName");

        final UUID id = tested.createOrder(product);

        verify(orderRepository).save(any(Order.class));
        assertNotNull(id);
    }
}
Copy
5. Application Layer

In this section, we'll implement the application layer. We'll allow the user to communicate with our application via a RESTful API.

So let's create the OrderController:

@RestController
@RequestMapping("/orders")
public class OrderController {

    private OrderService orderService;

    @Autowired
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    CreateOrderResponse createOrder(@RequestBody CreateOrderRequest request) {
        UUID id = orderService.createOrder(request.getProduct());

        return new CreateOrderResponse(id);
    }

    @PostMapping(value = "/{id}/products")
    void addProduct(@PathVariable UUID id, @RequestBody AddProductRequest request) {
        orderService.addProduct(id, request.getProduct());
    }

    @DeleteMapping(value = "/{id}/products")
    void deleteProduct(@PathVariable UUID id, @RequestParam UUID productId) {
        orderService.deleteProduct(id, productId);
    }

    @PostMapping("/{id}/complete")
    void completeOrder(@PathVariable UUID id) {
        orderService.completeOrder(id);
    }
}
Copy

This simple Spring Rest controller is responsible for orchestrating the execution of domain logic.

This controller adapts the outside RESTful interface to our domain. It does so by calling the appropriate methods from OrderService (port).

6. Infrastructure Layer

The infrastructure layer contains the logic needed to run the application.

We'll start by creating the configuration classes. First, we'll implement a class that will register our OrderService as a Spring bean:

@Configuration
public class BeanConfiguration {

    @Bean
    OrderService orderService(OrderRepository orderRepository) {
        return new DomainOrderService(orderRepository);
    }
}
Copy

Next, we'll create the configuration responsible for enabling the Spring Data repositories we'll use:

@EnableMongoRepositories(basePackageClasses = SpringDataMongoOrderRepository.class)
public class MongoDBConfiguration {
}
Copy

We've used the basePackageClasses property because those repositories can only be in the infrastructure layer. Hence, there's no reason for Spring to scan the whole application. Furthermore, this class can contain everything related to establishing a connection between MongoDB and our application.

Finally, we'll implement the OrderRepository from the domain layer. We'll use our SpringDataMongoOrderRepository in our implementation:

@Component
public class MongoDbOrderRepository implements OrderRepository {

    private SpringDataMongoOrderRepository orderRepository;

    @Autowired
    public MongoDbOrderRepository(SpringDataMongoOrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public Optional<Order> findById(UUID id) {
        return orderRepository.findById(id);
    }

    @Override
    public void save(Order order) {
        orderRepository.save(order);
    }
}
Copy

This implementation stores our Order in MongoDB. In a hexagonal architecture, this implementation is also an adapter.

7. Benefits

The first advantage of this approach is that we separate work for each layer. We can focus on one layer without affecting others.

Furthermore, they're naturally easier to understand because each of them focuses on its logic.

Advertisement: 0:07

Another big advantage is that we've isolated the domain logic from everything else. The domain part only contains business logic, and can be easily moved to a different environment.

In fact, let's change the infrastructure layer to use Cassandra as a database:

@Component
public class CassandraDbOrderRepository implements OrderRepository {

    private final SpringDataCassandraOrderRepository orderRepository;

    @Autowired
    public CassandraDbOrderRepository(SpringDataCassandraOrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public Optional<Order> findById(UUID id) {
        Optional<OrderEntity> orderEntity = orderRepository.findById(id);
        if (orderEntity.isPresent()) {
            return Optional.of(orderEntity.get()
                .toOrder());
        } else {
            return Optional.empty();
        }
    }

    @Override
    public void save(Order order) {
        orderRepository.save(new OrderEntity(order));
    }

}
Copy

Unlike with MongoDB, we now use an OrderEntity to persist the domain in the database.

If we add technology-specific annotations to our Order domain object, then we violate the decoupling between infrastructure and domain layers.

The repository adapts the domain to our persistence needs.

Let's go a step further and transform our RESTful application into a command-line application:

@Component
public class CliOrderController {

    private static final Logger LOG = LoggerFactory.getLogger(CliOrderController.class);

    private final OrderService orderService;

    @Autowired
    public CliOrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    public void createCompleteOrder() {
        LOG.info("<<Create complete order>>");
        UUID orderId = createOrder();
        orderService.completeOrder(orderId);
    }

    public void createIncompleteOrder() {
        LOG.info("<<Create incomplete order>>");
        UUID orderId = createOrder();
    }

    private UUID createOrder() {
        LOG.info("Placing a new order with two products");
        Product mobilePhone = new Product(UUID.randomUUID(), BigDecimal.valueOf(200), "mobile");
        Product razor = new Product(UUID.randomUUID(), BigDecimal.valueOf(50), "razor");
        LOG.info("Creating order with mobile phone");
        UUID orderId = orderService.createOrder(mobilePhone);
        LOG.info("Adding a razor to the order");
        orderService.addProduct(orderId, razor);
        return orderId;
    }
}
Copy

Unlike before, we've now hardwired a set of predefined actions that interact with our domain. We could use this to populate our application with mocked data, for example.

Even though we completely changed the purpose of the application, we haven't touched the domain layer.

8. Conclusion

In this article, we learned how to separate the logic related to our application into specific layers.

First, we defined three main layers: application, domain, and infrastructure. Then we described how to fill them, and explained the advantages.

Next, we came up with the implementation for each layer:

Finally, we swapped the application and infrastructure layers without impacting the domain.

The code backing this article is available on GitHub. Once you're logged in as a Baeldung Pro Member, start learning and coding on the project.

Azure Container Apps is a fully managed serverless container service that enables you to build and deploy modern, cloud-native Java applications and microservices at scale. It offers a simplified developer experience while providing the flexibility and portability of containers.

Of course, Azure Container Apps has really solid support for our ecosystem, from a number of build options, managed Java components, native metrics, dynamic logger, and quite a bit more.

To learn more about Java features on Azure Container Apps, visit the documentation page.

You can also ask questions and leave feedback on the Azure Container Apps GitHub page.
COURSES
ALL COURSES
BAELDUNG ALL ACCESS
BAELDUNG ALL TEAM ACCESS
THE COURSES PLATFORM
SERIES
JAVA "BACK TO BASICS" TUTORIAL
LEARN SPRING BOOT SERIES
SPRING TUTORIAL
GET STARTED WITH JAVA
SECURITY WITH SPRING
REST WITH SPRING SERIES
ALL ABOUT STRING IN JAVA
ABOUT
ABOUT BAELDUNG
THE FULL ARCHIVE
EDITORS
OUR PARTNERS
PARTNER WITH BAELDUNG
EBOOKS
FAQ
BAELDUNG PRO
TERMS OF SERVICE PRIVACY POLICY COMPANY INFO CONTACT
PRIVACY MANAGER
Skip Ad Testing with Spring and Spock

Last updated: January 15, 2024

Written by:
Jan Hauer
Reviewed by:
Tom Hombergs
Spring+Testing
Spock
1. Introduction

In this short tutorial, we'll show the benefits of combining the supporting power of Spring Boot's testing framework and the expressiveness of the Spock framework whether that be for unit or integration tests.

2. Project Setup

Let's start with a simple web application. It can greet, change the greeting and reset it back to the default by simple REST calls.  Aside from the main class, we use a simple RestController to provide the functionality:

@RestController
@RequestMapping("/hello")
public class WebController {

    @GetMapping
    public String salutation() {
        return "Hello world!";
    }
}
Copy

So the controller greets with 'Hello world!'. The @RestController annotation and the shortcut annotations ensure the REST endpoint registration.

3. Maven Dependencies for Spock and Spring Boot Test 

We start by adding the Maven dependencies and if needed Maven plugin configuration.

3.1. Adding the Spock Framework Dependencies with Spring Support

For Spock itself and for the Spring support we need two dependencies:

<dependency>
    <groupId>org.spockframework</groupId>
    <artifactId>spock-core</artifactId>
    <version>2.4-M4-groovy-4.0</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.spockframework</groupId>
    <artifactId>spock-spring</artifactId>
    <version>2.4-M4-groovy-4.0</version>
    <scope>test</scope>
</dependency>

Copy

Notice, that the versions are specified with are a reference to the used groovy version.

3.2. Adding Spring Boot Test

In order to use the testing utilities of Spring Boot Test, we need the following dependency:

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <version>3.2.1</version>
    <scope>test</scope>
</dependency>
Copy
3.3. Setting up Groovy

And since Spock is based on Groovy, we have to add and configure the gmavenplus-plugin as well to be able to use this language in our tests:

<plugin>
    <groupId>org.codehaus.gmavenplus</groupId>
    <artifactId>gmavenplus-plugin</artifactId>
    <version>3.0.2</version>
    <executions>
        <execution>
            <goals>
                <goal>compileTests</goal>
            </goals>
        </execution>
    </executions>
</plugin>
Copy

Note, that since we only need Groovy for test purposes and therefore we restrict the plugin goal to compileTest.

4. Loading the ApplicationContext in a Spock Test

One simple test is to check if all Beans in the Spring application context are created:

@SpringBootTest
class LoadContextTest extends Specification {

    @Autowired (required = false)
    private WebController webController

    def "when context is loaded then all expected beans are created"() {
        expect: "the WebController is created"
        webController
    }
}
Copy

For this integration test, we need to start up the ApplicationContext, which is what @SpringBootTest does for us. Spock provides the section separation in our test with the keywords like "when", "then" or "expect".

In addition, we can exploit Groovy Truth to check if a bean is null, as the last line of our test here.

5. Using WebMvcTest in a Spock Test

Likewise, we can test the behavior of the WebController:

@AutoConfigureMockMvc
@WebMvcTest
class WebControllerTest extends Specification {

    @Autowired
    private MockMvc mvc

    def "when get is performed then the response has status 200 and content is 'Hello world!'"() {
        expect: "Status is 200 and the response is 'Hello world!'"
        mvc.perform(get("/hello"))
          .andExpect(status().isOk())
          .andReturn()
          .response
          .contentAsString == "Hello world!"
    }
}
Copy

It's important to note that in our Spock tests (or rather Specifications) we can use all familiar annotations from the Spring Boot test framework that we are used to.

6. Conclusion

In this article, we've explained how to set up a Maven project to use Spock and the Spring Boot test framework combined. Furthermore, we have seen how both frameworks supplement each other perfectly.

For a deeper dive, have a look to our tutorials about testing with Spring Boot, about the Spock framework and about the Groovy language.

The code backing this article is available on GitHub. Once you're logged in as a Baeldung Pro Member, start learning and coding on the project.

Azure Container Apps is a fully managed serverless container service that enables you to build and deploy modern, cloud-native Java applications and microservices at scale. It offers a simplified developer experience while providing the flexibility and portability of containers.

Of course, Azure Container Apps has really solid support for our ecosystem, from a number of build options, managed Java components, native metrics, dynamic logger, and quite a bit more.

To learn more about Java features on Azure Container Apps, visit the documentation page.

You can also ask questions and leave feedback on the Azure Container Apps GitHub page.

COURSES
ALL COURSES
BAELDUNG ALL ACCESS
BAELDUNG ALL TEAM ACCESS
THE COURSES PLATFORM
SERIES
JAVA "BACK TO BASICS" TUTORIAL
LEARN SPRING BOOT SERIES
SPRING TUTORIAL
GET STARTED WITH JAVA
SECURITY WITH SPRING
REST WITH SPRING SERIES
ALL ABOUT STRING IN JAVA
ABOUT
ABOUT BAELDUNG
THE FULL ARCHIVE
EDITORS
OUR PARTNERS
PARTNER WITH BAELDUNG
EBOOKS
FAQ
BAELDUNG PRO
TERMS OF SERVICE PRIVACY POLICY COMPANY INFO CONTACT
PRIVACY MANAGER cloud.google.com では、サービスの提供および品質向上とトラフィックの分析に Cookie が使用されています。同意すると、広告の配信や、表示されるコンテンツと広告のパーソナライズにも Cookie が使用されます。 詳細。

同意する
同意しない
コンテンツに移動
Cloud
ブログ
お問い合わせ
無料で開始
デベロッパー
Google Cloud のデータベース ガイド: パート 3 - Cloud Spannerと Cloud Run での Spring Boot を使用した CRUD オペレーション
2022年7月26日
Google Cloud Japan Team

※この投稿は米国時間 2022 年 7 月 14 日に、Google Cloud blog に投稿されたものの抄訳です。

このブログで説明すること

Cloud Run にデプロイされた Cloud Spanner の DML API を使用して、Java Spring Boot アプリケーション上で CRUD のテストをします。Dockerfile は使用しません。このテストでは、ある住宅地域でのバドミントン コートの予約のユースケースを取り上げます。その理由は、私の住む地域では、利用できるすべてのコートを同じグループの人々が毎日占領してしまうという問題がありました。テストで紹介する予約システムを実装すると、このグループが予約できるのは 1 つのコートを 1 日 1 時間のみになり、予定をとった日だけしか使用できなくなるため、すべての人にコートを使用できる機会が公平に与えられるようになります。子供たちを含め、地域の人たちの利益になります。

Spanner を選ぶ理由

Spring Boot、Jib、Cloud Run での Cloud Spanner の実装の前に、まずは基本を確認してみましょう。Spanner は私の大好きなリレーショナル データベースで、次のような特徴を備えています。  

フルマネージド

ミッション クリティカルな RDBMS サービス

外部との整合性、原子性、独立性、耐久性のあるトランザクションを提供

業界屈指の 99.999% の可用性

マルチリージョン インスタンスのサポート

TrueTime 原子時計

透過的かつ同期的なレプリケーション

100% オンラインでのスキーマ変更とメンテナンス

ダウンタイムなしのトラフィック処理

上記すべてに加えてこの他もグローバル スケールで提供

すばらしいですね。一言では言い尽くせません。ここで頭が混乱しそうになるのもわかりますが、ご説明します。すべての特徴を一つひとつ説明したいのはやまやまですが、今回はこのブログのトピックに合わせて 2 つの特に優れた特徴を取り上げます。その他についてはプロダクトのページをご覧ください。

TrueTime

TrueTime は、すべての Google サーバー上のアプリケーションに提供される可用性の高い分散クロックです。

TrueTime により、アプリケーションは単調に増加するタイムスタンプを生成可能です。タイムスタンプ T が生成される前にタイムスタンプ T' が生成された場合、アプリケーションは T' より大きいことが保証された T を算出できます

この保証は、すべてのサーバーとすべてのタイムスタンプにわたって保持されます。Cloud Spanner はこの機能を使用してタイムスタンプをトランザクションに割り当てます

外部整合性

Cloud Spanner は、強整合性、線形化可能性、直列化可能性が統合された卓越したデータベース サービスだと言えるでしょう。

Cloud Spanner が実際にはパフォーマンスや可用性の向上のために複数のサーバー（場合によっては複数のデータセンター）でトランザクションを実行している場合でも、すべてのトランザクションが順次実行されるかのようにシステムが動作します

1 つのトランザクションが完了してから別のトランザクションの commit が開始する場合、クライアントは、2 番目のトランザクションの効果が反映されるという状態に遭遇することはありません

これらの優れた機能の詳細に関するドキュメントは、こちらとこちらでご覧いただけます。

では、そろそろ実装の詳細に入りましょう。ここでは、実装を次の 3 つのパートに分けて説明します。

Cloud Spanner の設定と DDL  

Spanner でのデータの変更

Cloud Run での Spring Boot と Cloud Spanner の手順

A. Spanner の設定と DDL

Cloud Spanner で CRUD オペレーションを行う前に、セルフペース ラボやドキュメントで、Cloud Spanner のインスタンス、データベース、テーブルの設定方法および基本的な DDL などを使用した操作方法の詳細を確認してください。

a. Google Cloud コンソールの [プロジェクト セレクタ] ページで、Google Cloud プロジェクトを選択または作成します

b. Cloud プロジェクトに対して課金が有効になっていることを確認します。詳しくは、プロジェクトで課金が有効になっているかどうかを確認する方法をご覧ください

c. プロジェクトに対して Cloud Spanner API を有効化します

d. インスタンスを作成します

e. インスタンス名に、「Test Instance」などの名前を入力します

f. インスタンス ID は、インスタンス名（「Test Instance」など）に基づき自動的に入力されます

g. デフォルトのオプション [リージョン] を保持し、プルダウン メニューから構成を選択します

h. インスタンスの構成により、インスタンスが保存および複製される地理的なロケーションが決まります

i. このテストの場合、[コンピューティング容量の割り当て] では 100 処理ユニットを設定できます

j. [作成] をクリックします。インスタンスがインスタンス リストに表示されます。

k. [Cloud Spanner インスタンス] ページに移動します

l. 作成したインスタンスをクリックして、[データベースの作成] をクリックします

m. DB 名、DB 言語を入力して [作成] をクリックします

n. データベースの [概要] ページの [テーブル] セクションで、[テーブルを作成] をクリックします

o. [DDL ステートメントの記述] ページで、次のように入力します

CREATE TABLE RESERVATION (
  ID STRING(70) NOT NULL,
  RESERVATION_DATE DATE NOT NULL,
  APT_ID STRING(50) NOT NULL,
  HOUR_NUMBER INT64 NOT NULL,
  PLAYER_COUNT INT64 NOT NULL
) PRIMARY KEY(ID);

p. [送信] をクリックすると完了です。このアプリには、トランザクション（予約情報）データを保持するテーブルが必要です

更新が完了すると、次のようなページが表示されます。

B. Cloud Spanner でのデータの変更

Cloud Spanner では次の 3 つの方法でデータを変更できます。

標準 DML

パーティション化 DML

ミューテーション

Cloud Spanner のデータ操作言語（DML）では、INSERT、UPDATE、DELETE のステートメントを使用してデータベース テーブルのデータを操作できます。DML ステートメントを実行するには、クライアント ライブラリ、Google Cloud コンソール、gcloud spanner を使用します。

標準 DML - 標準的なオンライン トランザクション処理（OLTP）ワークロードに適しています。
コードサンプルを含む詳細については、DML の使用をご覧ください。

パーティション化 DML - 以下の例のように、一括更新と削除用に設計されています。

定期的なクリーンアップとガベージ コレクション

デフォルト値での新しい列のバックフィリング

コードサンプルを含む詳細については、パーティション化 DML の使用をご覧ください

ミューテーション - 挿入、更新、削除など、データベース内のさまざまな行やテーブルに、Cloud Spanner によってアトミックに適用される一連の操作を表します。

1 つ以上の書き込みを含む 1 つ以上のミューテーションを定義したら、書き込みを commit するためにミューテーションを適用します

各変更は、ミューテーションに追加された順序で適用されます

コードサンプルを含む詳細については、ドキュメントをご覧ください

注: 今回の例では、Spring Boot フレームワークと Spring Data Cloud Spanner モジュールを使用しています。また、SpannerRepository インターフェースを拡張して、Cloud Spanner のデータをクエリおよび変更するすべてのアプリケーション ロジックをカプセル化するようにしています。このインターフェースは、DML クエリメソッドを使用して Cloud Spanner データで CRUD オペレーションを実行します。

C. Cloud Run での Spring Boot と Cloud Spanner

Spring Data Cloud Spanner モジュールは、Spring Framework で構築された Java アプリケーションで Cloud Spanner を使用する場合に役立ちます。

次の図は、このテストのアーキテクチャの概要を示しています。

1. Cloud Shell と Cloud Run の設定

Google Cloud はノートパソコンからリモートで操作できますが、ここでは Cloud Shell（Google Cloud 上で動作するコマンドライン環境）を使用します。

Cloud Shell をまだアクティブにしていない場合は、次の手順に沿って Cloud Shell をアクティブにし、認証が完了していること、プロジェクト ID（このブログのステップ A.1.a. で作成および選択済み）に設定されていることを確認してください。

なんらかの理由でプロジェクトが設定されていない場合は、次のコマンドを実行します。

gcloud config set project <PROJECT_ID>

Cloud Shell から次の Cloud Run API を有効にします。
gcloud services enable run.googleapis.com




注: プロジェクトをブートストラップせず、次のステップを実行しない場合は、Cloud Shell で次のコマンドを実行してプロジェクト リポジトリのクローンを作成できます。

git clone https://github.com/AbiramiSukumaran/spanner-example.git

git clone https://github.com/AbiramiSukumaran/springboot-client.git




2. Spring Boot Java サーバーアプリ（REST API）のブートストラップ

Cloud Shell 環境から次のコマンドを使用して、新しい Spring Boot アプリケーションを初期化およびブートストラップします。

$ curl https://start.spring.io/starter.tgz -d packaging=jar -d dependencies=cloud-gcp,web,lombok -d baseDir=spanner-example -d bootVersion=2.3.3.RELEASE | tar -xzvf -

$ cd spanner-example

リポジトリのクローンを作成しない場合は、このコマンドを使用してください。このコマンドにより、Maven の pom.xml、Maven ラッパー、アプリケーションのエントリポイントとともに、新しい Maven プロジェクトを含む新しい spanner-example/ ディレクトリが作成されます。

pom.xml ファイルに Spring Data Cloud Spanner スターターと必要な他の依存関係を追加します。spanner-example/pom.xml
. . .
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
 
        <!-- Add Spring Cloud GCP Spanner Starter -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-gcp-starter-data-spanner</artifactId>
            <version>1.2.8.RELEASE</version>
        </dependency>
 
        <!-- https://mvnrepository.com/artifact/org.projectlombok/lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.24</version>
            <scope>provided</scope>
        </dependency>
 
    </dependencies>
. . .

application.properties で Spanner データベース接続情報を構成します。

spanner-example/src/main/resources/application.properties

アプリをビルドします。./mvnw package

../spanner-example/src/main/java/com/example/demo/Reservation.java でエンティティ クラスを作成 - Spring Cloud GCP の Spring Data Spanner サポートにより、Spring Data を使用して Java オブジェクトと慣用的な ORM マッピングを Spanner テーブルに簡単に作成できます。
今回は、スキーマ設計のベスト プラクティスを確認するとともに、サーバーのワークロードの分散におけるホットスポットを回避する主キーを選択しました。
@Column(name="APT_ID")
  private String aptId;
 
  @Column(name="HOUR_NUMBER")
  private int hourNumber;
 
  @Column(name="PLAYER_COUNT")
  private int playerCount;
}

次のコンテンツを含む ReservationRepository クラスを作成します。
spanner-example/src/main/java/com/example/demo/ReservationRepository.java

インターフェースは、Reservation がドメインクラスで String が主キータイプの SpannerRepository<Reservation, String> を拡張します。Spring Data はこのインターフェースを通じて自動的に CRUD アクセスを提供するため、追加のコードを作成する必要はありません。

  基本オペレーション（挿入、更新、削除、検索、ID で検索、条件で検索）のための REST Controller を以下のファイルの ReservationController クラスに作成します。../spanner-example/src/main/java/com/example/demo/DemoApplication.java

@RestController
class ReservationController {
  private final ReservationRepository reservationRepository;
 
  ReservationController(ReservationRepository reservationRepository) {
            this.reservationRepository = reservationRepository;
        }
 
    //ID を指定して予約を読み取り
    @GetMapping("/api/reservations/{id}")
    public Reservation getReservation(@PathVariable String id) {
        return reservationRepository.findById(id)
                            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, id + " not found"));
    }
 
    //パラメータ化したチェックを行い予約を読み取り
    @GetMapping("/api/getreservations/{id}")
    public String getReservations(@PathVariable String id) {
        String dateId = id.split("_")[0];
        String hourId = id.split("_")[1];
        Iterator<Reservation> iterator = reservationRepository.findAll().iterator();
        String element = "";
        String date = "";
        String hour = "";
        while (iterator.hasNext()) {
            Reservation res = iterator.next();
            date = res.getId().toString().split("_")[1];
            hour = Integer.toString(res.getHourNumber());
            if(date.equals(dateId) && hour.equals(hourId)){
                return "present";
            }
        }
        return "absent";                
    }
   
    //予約を挿入
    @PostMapping("/api/reservation")
    public String createReservation(@RequestBody Reservation reservation) {
            java.util.Date dt = new java.util.Date();
            java.util.Calendar c = java.util.Calendar.getInstance();
            c.setTime(dt);
            dt = c.getTime();
            Date d = Date.fromJavaUtilDate(dt);
            reservation.setId(reservation.getAptId() + "_" + d);
            Reservation saved = reservationRepository.save(reservation);
        return saved.getId();
    }
 
    //ID を指定して予約を更新
    @PutMapping("/api/{id}")
    public Reservation updateReservation(@RequestBody Reservation reservation, @PathVariable ("id") String id) {
        Reservation existingReservation = this.reservationRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, id + " not found"));
        existingReservation.setAptId(reservation.getAptId());
        existingReservation.setHourNumber(reservation.getHourNumber());
        existingReservation.setPlayerCount(reservation.getPlayerCount());
         return this.reservationRepository.save(existingReservation);
    }
   
    //ID を指定して予約を削除
    @DeleteMapping("/api/{id}")
    public ResponseEntity<Reservation> deleteReservation(@PathVariable ("id") String id){
        Reservation existingReservation = this.reservationRepository.findById(id)
                    .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, id + " not found"));
         this.reservationRepository.delete(existingReservation);
         return ResponseEntity.ok().build();
    }}

アプリケーションを再ビルドして実行します。
./mvnw package
./mvnw spring-boot:run

3. Docker なしでのアプリのコンテナ化

Jib により、Dockerfile / デーモンなしでアプリを最適な方法でコンテナ化し、任意の Container Registry に公開できます

次に進む前に、Container Registry API をアクティブにする必要があります。これは、API にアクセスできるようにするために各プロジェクトに 1 回のみ行う必要があります

$ gcloud services enable containerregistry.googleapis.com

Jib を実行して、Docker イメージをビルドして Container Registry に公開します

$ ./mvnw com.google.cloud.tools:jib-maven-plugin:3.1.1:build \

 -Dimage=gcr.io/$GOOGLE_CLOUD_PROJECT/<<your-container-name>>

注: このテストでは Jib Maven プラグインを pom.xml で構成していませんが、高度な使用の場合は、より多くの構成オプションのあるプラグインを pom.xml に追加することもできます

Google Cloud コンソールに移動し、ナビゲーション メニューをクリックして [Container Registry] を選択し、イメージが適切に公開されていることを確認します

4. Cloud Run へのデプロイ

次のコマンドを実行して、Cloud Run にコンテナ化されたアプリをデプロイします。

gcloud run deploy <<application>> --image gcr.io/$GOOGLE_CLOUD_PROJECT/<<container>> --platform managed --region us-central1 --allow-unauthenticated --update-env-vars DBHOST=$DB_HOST

–allow-unauthenticated で、認証なしでサービスを届けられるようになります。

–platform-managed とは、Anthos を介した Kubernetes のものではなく、フルマネージド環境をリクエストしていることを意味します。

–update-env-vars では、接続文字列は、環境変数「DBHOST」に渡されることになります。

デプロイが完了したら、コマンドラインにデプロイしたサービスの URL が表示されます。

サービスの URL に接続すると、ブラウザにウェブページと、Cloud Logging の [ログ エクスプローラ] ページにログが表示されます。




生成した Cloud Run URL で REST API にアクセスできるようになりました。

5. Spring Boot Java クライアント アプリのブートストラップ（予約ユーザー インターフェース）

REST API のサーバー アプリケーションと同様に、この Spring Boot フレームワークのクライアント アプリケーションの構造は、リポジトリのクローン作成後は次のようになります。




次の Cloud Shell コマンドを使用してクライアント アプリケーションをブートストラップすることもできます。

$ curl https://start.spring.io/starter.tgz -d packaging=jar -d dependencies=cloud-gcp,web,lombok -d baseDir=springboot-client -d bootVersion=2.3.3.RELEASE | tar -xzvf -

デモフォルダには DemoApplication クラス、Controller クラス、Bean クラスが以下の場所に含まれます。
../springboot-client/src/main/java/com/example/demo/DemoApplication.java
../springboot-client/src/main/java/com/example/demo/MyController.java
../springboot-client/src/main/java/com/example/demo/Reservation.java

対応する github ソースリンクは次のとおりです。

https://github.com/AbiramiSukumaran/springboot-client/blob/main/src/main/java/com/example/demo/DemoApplication.java

https://github.com/AbiramiSukumaran/springboot-client/blob/main/src/main/java/com/example/demo/Reservation.java

MyController.java クラスには、サーバー アプリケーションで作成した REST API を呼び出すメソッド、CRUD HTML ページにルーティングするメソッド、サーバーサイドの検証を実行するメソッドが含まれます。

https://github.com/AbiramiSukumaran/springboot-client/blob/main/src/main/java/com/example/demo/DemoApplication.java

1. 該当日にそのユニットに既存の予約があるかどうかを検証する API を呼び出すメソッド

validateId(Reservation newReservation)

2. 別のユニットによってその時間がすでに予約されているかどうかを検証する API を呼び出すメソッド

validateSlot(Reservation newReservation)

3. 特定の予約を取得する API を呼び出すメソッド

callReservationsByIdAPI(Reservation reservation)

4. 予約表示の呼び出しで呼び出され、showMessage HTML ページを返すメソッド

showForm(Reservation reservation

5. 検索で呼び出され、searchReservation HTML ページを返すメソッド

searchForm(Reservation reservation)

6. ホームページで呼び出され、HomePage HTML ページを返すメソッド

homeForm(Reservation reservation)

7. CRUD 呼び出しのメソッド

sendForm(Reservation reservation)

processForm(Reservation reservation)

editForm(Reservation reservation)

deleteForm(Reservation reservation)

Thymeleaf は、ウェブ環境とスタンドアロン環境の両方に対応するサーバーサイド Java テンプレート エンジンです。その主な目的は、開発ワークフローに優れたテンプレートを実現することです。つまり、ブラウザに適切に表示され、静的プロトタイプとしても機能する HTML です。これにより、開発チームとのコラボレーションが向上します。

../templates フォルダには、CRUD HTML ページ（ビューレイヤ）用の Thymeleaf テンプレートが以下の場所に含まれます。
../springboot-client/src/main/resources/templates/
GitHub ソースリンク:
https://github.com/AbiramiSukumaran/springboot-client/tree/main/src/main/resources/templates

このビューレイヤには、クライアント側の検証のためのメソッドも含まれます。

null 以外のフィールドを検証します

予約番号、時間数、プレーヤー人数のデータ形式が適切かどうか入力を検証します

サーバー アプリケーションの pom.xml コンテンツの他に、クライアント アプリケーションの Thyme 依存関係を追加する必要があります

      <dependency>

           <groupId>org.springframework.boot</groupId>

            <artifactId>spring-boot-starter-thymeleaf</artifactId>

        </dependency>

以下のステップについては、残りはこのリストのサーバー アプリケーション（セクション C、ポイント 2）と同様です。

ビルドと実行
./mvnw package
./mvnw spring-boot:run

Docker なしでの Jib を使用したコンテナ化
$ ./mvnw com.google.cloud.tools:jib-maven-plugin:3.1.1:build -Dimage=gcr.io/$GOOGLE_CLOUD_PROJECT/<<your-container-name>>

Cloud Run でのデプロイ
gcloud run deploy <<application>> --image gcr.io/$GOOGLE_CLOUD_PROJECT/<<container>> --platform managed --region us-central1 --allow-unauthenticated --update-env-vars DBHOST=$DB_HOST

アプリはクラウドに送られるため、ログをご確認ください。デプロイが完了したら、クライアント アプリに URL が表示されます。

URL を開いてアプリケーションで CRUD を実行します。この動画は、選択型のクライアント サイドおよびサーバーサイドの検証による、バドミントン スロットの予約の作成、既存の予約の検索、既存の予約の編集、既存の予約の削除のデモです。

まとめ

Cloud Spanner は、使用の増大に合わせて容易にスケーリングできるフルマネージド リレーショナル データベースを探している企業にとって最適な選択肢です。「処理ユニット（PU）」と呼ばれるきめ細かなインスタンス コンピューティング容量により、通常のインスタンスのわずか 10 分の 1 のコスト（毎月約 $65）で、Spanner でワークロードを実行できます。Spanner のすべての機能とサンプル ユースケースをぜひご覧ください。PostgreSQL 構文に慣れている場合は、PostgreSQL Interface もご利用いただけます。

まとめ

Cloud Run にデプロイされた Docker を使用しないコンテナによる、Spring Boot での Cloud Spanner の簡単なテストはお楽しみいただけたでしょうか。今回のテストから以下の検証を意図的に省いています。練習用として実装してみてください。

スロットが満席のときに新しい予約が作成されないようにするにはどうしたらいいでしょうか。

シングルスとダブルスの選択を尋ねてそれに応じてチームを作成できるように、アプリケーションを拡張するにはどうしたらいいでしょうか。




以下の　Codelab を実装にぜひお役立てください。

https://codelabs.developers.google.com/codelabs/cloud-spanner-first-db#0

https://codelabs.developers.google.com/codelabs/cloud-spring-spanner#0

https://codelabs.developers.google.com/codelabs/cloud-kotlin-jib-cloud-run#4

https://codelabs.developers.google.com/codelabs/cloud-run-hello#4




また、LinkedIn でアイデアやご意見・ご感想をぜひお寄せください。


Google、デベロッパー アドボケイト Abirami Sukumaran
投稿先
デベロッパー
データベース
関連記事
AI & Machine Learning
MCP を使用した ADK エージェントを A2A フレームワークに変換する方法

執筆者: Neeraj Agrawal • 所要時間: 3 分

Developers & Practitioners
エージェントを作り上げるツール: ADK を使用してゼロからアシスタントを作成

執筆者: Jack Wotherspoon • 所要時間: 12 分

Developers & Practitioners
わずか 10 分でリモート MCP サーバーを構築して Google Cloud Run にデプロイ

執筆者: Jack Wotherspoon • 所要時間: 7 分

Developers & Practitioners
Gemini CLI : オープンソース AI エージェント

執筆者: Ryan J. Salva • 所要時間: 2 分

フッターのリンク
フォロー
Google Cloud
Google Cloud プロダクト
プライバシー
規約
Cookie を管理
ヘルプ
Language
‪English‬
‪Deutsch‬
‪Français‬
‪한국어‬
‪日本語‬ 