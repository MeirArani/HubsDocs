# 初めに
Hubs改善には様々な問題に遭遇しますが、途中で私はコードベースをよく知るようになって、有意義な変更（例えば、ビルド加速・開発者体験・現代的なツール対応）を加えています。  
そこで、Hubsの物理エンジンとレンダリングのアップグレードについて詳しく説明したいと思います。
# 構造総括
Hubsというのはひとつのパッケージではなく、全体は何百ものパッケージからできています。大部のパッケージは簡単にアップグレードできるのに、hubsの基本的な実行に関わるパッケージはじっくり修理しなければなりません。

物理エンジンについては：　

- `ammo.js`: 「Bullet」というC＋＋エンジンに基づいて物理エンジンのメインパッケージ
- `three-to-ammo`: `ammo.js` と`three`の違いを埋める実用的なパッケージ
- `ammo-three`: 3Dオブジェクトと衝突形状の作成・削除・アップデートと同時にできるライブラリー
- `ammo-debug-drawer`:物理エンジンのデバッグツール

# `ammo.js`
`ammo.js`というのは「Bullet」というC＋＋物理エンジンを基にした派生的なJavascriptパッケージです。`Emscripten`というツールで、JS機能を使って元のC＋＋コードが実行できます。しかし、C++とJSの大きな違いのせいで、そのような言語間の通信が大変な問題を伴います。

元の`ammo.js`は3年以上アップデートされていません。hubs向けの新しい機能が含まれたHubs Foundationのフォークも何年か遅れています。

そこで私は元のBullet物理エンジンとEmscriptenを数日間勉強しました。ついに新しい`ammo.js`ビルドが作成できました。含まれなかった現代的な機能が少しありますが、必要な時にはEmscriptenに自分で実装できます。

さらにTypescriptサポートを追加しました。

# `three-to-ammo`
`three-to-ammo`は`three` (３Ｄレンダリングエンジン)と`ammo.js`　(物理エンジン)の互換性レイヤーです。その二つは全く無関係のエンジンですので、ユーザが自分で同期させる必要があります。つまり、物理オブジェクトと3Dモデルの動きを一つ一つ同期させないと、ソフトが故障して、クラッシュするはずだ。

hubs開発者が作ったパッケージですが、数年間アップデートされませんので、私はフォークして、自分で修理しました。

`three-to-ammo`の主な機能は、3D形状に基づいて衝突形状の作成です。ユーザーは`createCollisionShape`という機能に3Dモデルデータを入力して、衝突形状（`btCollisionShape`）が出力されます。

一目見ただけで簡単なシステムに見えるが、Typescript化は思ったより難しくなりました。Javascriptと違って、Typescriptではビルド前に厳しいルールが課せられて、開発する時にバグを早く認めますので、プログラマーの観点から言うと有益なツールです。

つまり、Javascriptなら、Typescriptを使用すればいいと考えられています。しかし、Emscripten（つまりJavascriptだけではなく、C＋＋も）使う場合には、Typescriptの強みが弱みになります。それで、Typescriptの意外な問題を自分で解決する必要がありました。

処理することには時間がかかりましたが、分かりやすくて、テストできて、効率的なシステムを実装できました。

# `three-ammo`
次は3Dと物理データをアップデート・削除できる`three-ammo`のパッケージです。

`three-ammo`には特別な「Web Worker」というシステムがあります。Web Workerというのは現代的なウェブブラウザで実行できるマルチスレッド・プロセスです。ウェブサイトのメイン・プロセスと同時に実行できるWeb Workerで、時間のかかるタスク（DBアクセスや、3Dと物理エンジン計算など）がメイン・プロセスを止めずに進んでいます。

ゲームと３Ｄソフトの視点から、Web Workersは有益なツールだと思われていますが、以外のプロセスですので、Javascriptの基本的な機能を半分ぐらい利用できなくて、特別な実装方法の必要もあって、IPC（プロセス間通信）で同期が合わないエラーや、デバッグしにくいエラーも生み出して、複雑なツールです。

`ammo.js`のC＋＋コードに組み合わせると、問題がたくさん起こります。不明瞭なバグと、自動的に逆コンパイルしたC＋＋コードだけでデバッグするしか方法はありません。

## メモリの困り
最も大変な問題はこれでした。理解できない理由で、基本的な論理は、予測できない状態になりました。先ほど動けた機能が偶然にクラッシュして、それなのにすぐ後でもう一回実行すると問題なしという状態です。

几帳面なデバッグした後で、ついに直せました。

原因は`Emscripten`が実行時に`ammo.js`を作成する方法です。普通はJavascriptがインポートしたコードを一回だけ実行するシステムですが、Emscriptenの場合はインポートする時に何回もコードを実行できます。それで、ファイル当たり特有`ammo.js`のコピーを受け取ります。

例えば、File AでFile Bにある機能をx変数を入力して実行すると、プログラム・クラッシュの可能性があります。それからFile Aにあるxという数字はFile Aのメモリに存在し、無関係のFile Bのメモリにアクセスできなくて、クラッシュするはずです。それで、デバッグする時に分析しようとした変数が理由なく消えて、偶然再び現れました。

そこで、私は一回だけ作成できる共有メモリのシステムを`ammo.js`に加えました。

## Web Workerの改善
上記の通り、Web Workersは色んなところからプロジェクトに複雑さを増す傾向があって、コードベースの重要な部分を何度も書き直す必要がありました。

具体的に、`SharedArrayBuffer`という技術を活かして、前よりも効率的なWeb Workerが作れました。

例えば、下記の実験結果をご覧ください。

**普通の`ArrayBuffer`で１０００個ボールの生成（５〜７ＦＰＳ）**

**`SharredArrayBuffer`で１０００個ボールの生成（〜６０ＦＰＳ）**

# Ammo Debug Drawer
`AmmoDebugDrawer`というのは、`three-to-ammo`と`ammo-three`のデバッグするためのサポートパッケージです。上記のアップグレードした後で、簡単に改善しました。

# 雑改善
## Vite
TypeScript化や、コードの近代化や、不必要なパッケージを削除することの上に、私はhubsのビルドと開発サポートシステムをWebpackからViteに切り替えました。Webpackに比べて、Viteははるかに使いやすくて効率的なソフトだと思います。Viteのv7.0はもうWebpackより何倍も速いことのさらに、新しいv8.0はビルドを最大30倍速くできます。
## Vitest
上記の通りに、Vitestというテストフレームワークを`three-to-ammo`に実装しました。Vitestは最初からVite向けのフレームワークですので、同じメリットがあります。テストフレームワークで、新しいコードの結果とバグの出現を数秒以内に確認できます。
## Yarn / Monorepoの構成変更
最後には、`npm`というパッケージ管理システムから、開発した`yarn`への変更について述べます。Yarnは便利な機能をたくさん備えていて、npmより効率的なパッケージ管理システムです。例えば、Yarnではシステム構造のエラーや、パッケージ構造のエラーを実行する前にも警告できて、開発者にデバッグの時間と手間を省けます。

それに、Yarnでは繋がっているパッケージを一緒に管理できる「Workspaces」という機能も活かせます。Workspacesで全てのhubsパッケージ（現在、〜１０個があります）を一緒に管理して、共有な依存パッケージをコピーを生成せずに共有できます。

Workspacesは開発する時に役に立ちます。例えば、`ammo.js`のコードをアップデートすると、新しいビルドを手動に実行せずに、`three-ammo`にすぐに使えます。hubsの開発は数パッケージと同時に開発することですので、Workspacesはもう何時間を省きました。

他のWorkspace（Monorepoの別名でも知られる）を利用するメリットを今後のアップデートで説明します。

# 結論
要するには、hubsの基本的な３Ｄレンダリングと物理エンジンのパッケージを改善して、YarnとViteの変更で開発者体験を改良して、今後の開発を合理化しました。

今からA-FrameのTypeScript化を完成させて、来月に発表できるプロトタイプを作成するつもりで進んでいます。

# 英語版
# Introduction 
I’ve ran into a lot of issues while trying to improve hubs, which has caused a lot of delays and unexpected issues. The good news is that I have been improving my understanding of the code base while making meaningful changes to the system—significantly hastening build times, improving the developer experience, and improving compatibility with modern tools and devices. 

Most recently, I performed a full upgrade of hubs physics and rendering systems. Allow me to share some details and results.

# Structural Overview
Hubs isn’t just one package or even a few packages—the entire system is comprised of hundreds and hundreds of dependent code bases. Thankfully, most packages can be used and upgraded with little worry. There are  however some packages which are critical to hubs’ basic operation—meaning an upgrade of hubs requires heavy updates to these packages as well. 

In the case of the physics engine, there are again several packages: 
- `ammo.js`: The main physics engine, derived from the C++ Bullet engine
- `three-to-ammo`: A utility package that bridges some differences between `threejs` (the 3D rendering engine) and `ammo.js` (the physics engine)
- `ammo-three`: A collection of tools to create, destroy, and update shapes and objects for the 3D engine and physics engine.
- `ammo-debug-drawer`: A utility tool for visualizing and debugging the physics engine. 

# `ammo.js`
`ammo.js` is a javascript derivative of the Bullet Physics Engine—originally a C++ library. The original code base is compiled from C++ into Javascript via a tool named `Emscripten`. This allows users to call functions using javascript which ‘communicate’ with the original C++ library. Since C++ and Javascript are very different languages, this cross-language communication is very difficult and fraught with issues.

The original `ammo.js` library hasn’t been updated in over three years. The Hubs Foundation created their own fork of the project, adding custom features and additional rendering functions. The hubs fork was also several years out of date. 

The first thing I did was study ammo, the original Bullet physics library, and how exactly I could use emscripten to compile from C++ into Javascript. I faced many complex difficulties along the way, which delayed me for many days. 

Eventually I was able to reliably compile a javascript version of the engine with many modern features. Unfortunately some features are still disabled (as Emscripten does not support them yet), but I may consider contributing those features to Emscripten if the need arises. 

I also added Typescript support to the library. This was a very difficult task as well. It’s not a perfect implementation, but it works well enough for our purposes.

# `three-to-ammo`
`three-to-ammo` primarily serves as a compatibility layer between `three` (the 3D rendering engine) and `ammo.js` (the physics engine.) Since the two are entirely unrelated, the user has to manually synchronize the two. This means every movement of a physics object in `ammo.js` requires the same movement of the 3D object in `three`. If anything goes out of sync, the entire program is at risk of crashing.

This library was originally written by one person working with the hubs team. The system is critical to hubs’ basic functions, but hasn’t been updated for years, so I forked my own copy to work on.

`three-to-ammo`’s main feature is the creation of collision shapes based on 3D models. The user can call functions like `createCollisionShape`—which takes a `three` 3D model as input,  and returns a new physics engine `btCollisionShape` as output.

Redesigning this library for typescript proved to be very difficult. Typescript is a great system because it requires strict rules while programming. This means it’s easy to catch bugs and errors *while* you are writing code—since breaking Typescript’s strict rules is the same as writing a bug in Javascript. Typescript will not allow a bug to compile, while Javascript checks almost nothing before executing code.

The issue with Typescript, is when you *aren’t* using Javascript. In this case, we’re calling functions in *C++*, not Javascript. This means I had to handle a lot of special cases that you don’t often find while programming Typescript code. 

I spent a lot of time solving these issues, but I’ve managed to create a system that is easy to understand, efficient, modern, and quickly testable. I added in a test suite as well—making it easy to check for issues with just a single button press. [Insert TestRunner here] 

# `three-ammo`
Next was `three-ammo`, which handles active tasks like creating, updating, and deleting the 3D/Physics data. It was developed by the same person who created `three-to-ammo`. 

`three-ammo` includes a special Web Worker class. Web Workers are a modern feature of web browsers which allow special code to run on a thread separate from the main program. This means that large and complex tasks (like loading or calculating massive data sets) can be done *concurrently*, which improves performance and reduces lag. This is key for a task like updating every single 3D model and physics object in a software like hubs.

Unfortunately, Web Workers are also very complex. They don’t have access to much of Javascript’s basic functionality, and have to be coded in a very unique way. The main program and Web Worker communicate via special “messages”—meaning communication timing issues and small errors in code become hard to trace.

This, combined with ammo’s C++ code, caused a lot of headaches for me. Errors were vague and nearly impossible to debug. When something didn’t work, I would be left with nothing but the decompiled C++ source (which usually reads like assembly code) to debug with. 

## Memory Woes

This set me back by several days. Random functions and basic logic seemed to be unpredictable, usually for reasons beyond comprehension. I eventually solved the issue after slow and careful debugging. 

The problem turned out to be caused by how ammo was instantiated at runtime. Usually, javascript code is defined *once* per execution—if several files reference a function or object, they are referencing the *same* one. Emscripten handles C++ code differently. It turns out that each JS file was given its own *unique* copy of the *entire* physics engine. This means that `file A` and `file B` had entirely different, very large memory heaps allocated to them! 

If `file A` called a function in `file B` and passed along a variable as input, the program would crash in unpredictable ways. This is because the variable was stored in `file A`’s memory, while `file B` had no reference to the variable. Debugging was difficult because variables would appear and disappear for seemingly no reason. That is, until I eventually realized every single file had its own version of ammo running independently! 

I added custom code to `ammo.js` to force a singular reference to the physics engine—meaning all code, no matter the package, is accessing the same physics engine. 

## Worker Improvements
As I mentioned above, Web Workers also add a lot of complexity to a project. Communication between the worker and the main code is very fragile and breaks easily. This required several re-writes to critical parts of the code base. 

Despite this, I managed to successfully produce a modern, Typescript based Web Worker that is far more performant. This was accomplished mainly through the use of `SharedArrayBuffer`—a modern Javascript feature that allows certain data to be read and modified in very efficient ways. 

For example, take a look at this comparison.

(Spawning 1000 balls using a regular `ArrayBuffer`: 5-7 FPS)

(Spawning 1000 balls using a `SharedArrayBuffer`: ~60 FPS)

# AmmoDebugDrawer
This package is an extension of the other two. It basically allows for additional features in `three` and `ammo` that aid debugging. After fixing the other two packages, this one was luckily very easy to improve. 

# Other Improvements 
## Vite
On top of modernizing the code, transitioning to Typescript, and removing unused packages, I also switched the build and development system from Webpack to Vite. Vite is a far more modern solution that is simpler, faster, and dynamic. `v7.0` of Vite was already several times faster than Webpack, but the new `v8.0` is a major improvement on top of that—boasting up to a ***30x*** speedup compared to Webpack. 
### Vitest
As I mentioned above, I also incorporated testing into the `three-to-ammo` package, which improves debugging and testing for new features. I decided to use Vite’s own `Vitest` package for this, which incorporates well with vite and makes uses of the same features.
## Yarn / Monorepo Configuration
I’ve also mentioned the change from the standard `npm` node system to `yarn` in the previous write-up. Yarn is far more efficient and provides many useful features that `npm` lacks. Yarn can spot issues in code automatically and warn the user before the program even executes—saving me a lot of headaches when developing.

Yarn also makes use of a system called *Workspaces*, which allows you to collect several related packages into one special group. As I mentioned above, Hubs includes several connected packages, which makes it a perfect fit for yarn and workspaces. 

For example, I can update code in `ammo.js` and have it immediately appear in `three-ammo`, which saved me hours of waiting for new builds after changing just one line of code. 

On top of this, yarn improves build times and storage usage by *collecting* the dependencies shared between packages in a workspace. If `three-ammo` and `ammo.js` both depend on `typescript`, then yarn will install *one* copy of `typescript` that is shared *between* the two, which is a massive help. 

There are many other benefits to using a workspace (otherwise called a “monorepo”), which I hope to go over in a future update. 

# In Summary
To summarize this report—I have successfully updated the core packages which hubs uses for physics and 3D rendering. These packages are very unique and presented many difficulties, but I managed to overcome the majority of issues given enough time. I have also made great strides at improving the general developer experience for the entire hubs ecosystem vita the change to Yarn and Vite.

I will continue my work by finishing A-Frame’s Typescript conversion. After this, I should hopefully be able to produce a working prototype after some bugfixing. 
