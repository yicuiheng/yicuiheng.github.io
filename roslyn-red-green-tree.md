# Roslyn の実装にみる Red Green Tree

> 「ジャングルは、最初の都市ができる前からここにある。最後の都市が滅んだ後にもあるでしょう。」

（[《老樹林のドライアド》](https://mtg-jp.com/products/card-gallery/0000035/435355/) のフレーバーテキストより）

最近 Roslyn のコードを読んでいて Red Green Tree というものを知ったので、それについて書いてみます。

昨今、Language Server Protocol によって開発支援機能が規格化されたのもあって、世の開発環境がどんどんリッチになっています。
それの伴って言語処理系側に対する要求も増えており、特に次の2つは開発支援機能を実装する上で重要です。
- 修正途中の不完全なソースコードを入力として与えられて、ユーザにとって有益な情報を出力する
- 前回からのソースコードの修正差分が与えられて、構文木を更新する

これらはテキストエディタ上でソースコードが修正される度に実行されユーザが不快にならない時間内に処理を完了する必要があります。
そういった要求を満たすために考えられた構文木の設計の一つが Red Green Tree です。
はじめは Roslyn で実装され、のちに Swift や Rust の言語サーバーである rust-analyzer でも使われるようになりました。

## Red Green Tree の概要

Red Green Tree のアイデアは構文木を Green Tree と Red Tree というレベルが異なる 2 つの木で表現するというものです。
Green Tree は構文解析によって生成される木で、Red Tree は Green Tree に対する操作機能を提供するための木です。

### Green Tree
Green Tree は空白やコメントを含めてソースコードから得られるすべての情報を保持しているので、後で元のソースコードを復元することができます。これのおかげでユーザフレンドリーなエラー報告を実現できています。
Green Tree の特殊な点は、ソースコード上での位置ではなくソースコード上での長さの情報を保持していることです。これによってソースコードの修正差分を基に構文木を更新するコストを減らせます。
具体例を使って説明します。以下のプログラムを考えましょう。

```csharp
int Id(int x)
{
    return x;
}
```

これを構文解析すると以下のような Green Tree を得ます。後ろにカッコつきで付与している数字はそのノードの長さです。また "\n" は実際のソースコード中での表現に依らず改行の意です。

FunctionDeclaration (32)
- "int " (4)
- "Id" (2)
- Params (8)
  - "(" (1)
  - "int " (4)
  - "x" (1)
  - ")\n" (2)
- "{\n    " (6)
- Statements (10)
  - ReturnStatement (10)
    - "return " (7)
    - "x" (1)
    - ";\n" "2"
- "}\n" (2)

ここで範囲 (27-28) を `42` に書き換える (つまり `return x;` を `return 42;` に変える) という修正差分を受け取った時、更新後の Green Tree は次のようになります。

FunctionDeclaration (33)
- "int " (4)
- "Id" (2)
- Params (8)
  - "(" (1)
  - "int " (4)
  - "x" (1)
  - ")\n" (2)
- "{\n    " (6)
- Statements (11)
  - ReturnStatement (11)
    - "return " (7)
    - "42" (2)
    - ";\n" "2"
- "}\n" (2)

ここで注意して欲しいのは、更新が必要なノードは "x" とその祖先ノードたち (ReturnStatement, Statements, Params, FunctionDeclaration) だけという点です。
もし Green Tree が長さではなくソースコード上の位置情報を保持していれば、"x" 以降のトークンの位置がずれるのでそれらの更新も必要になっていたことでしょう。

### Red Tree
Red Tree は Green Tree への参照を持ち、構文木に対する様々な操作をする API を提供します。
Red Tree の特殊な点は子ノードの生成が遅延されるということです。

Green Tree の時に使用した例を使って Red Tree がどのように構築されるかを説明します。
まず 1 つめの入力に対して以下のような Red Tree が得られます。後ろにカッコつきで付与している (a-b) はそのノードがソースコード上で a バイト目から b バイト目にあることを意味しています。

FunctionDeclaration (0-32)

Green Tree トップレベルの要素に対応する Red Tree のノードしか作られません。
次に Green Tree の時と同様の修正差分が与えられた時、Red Tree は Green Tree を更新するために次の木を作ります。

FunctionDeclaration (0-32)
- Statements (20-30)
  - ReturnStatement (20-30)
    - "x" (27-28)

これによって修正すべき Green Tree のノードは "x" と ReturnStatement, Statements, FunctionDeclaration だと分かります。
今回は修正箇所が小さかったので木というよりはパスになりましたが、複数のトークンにまたがる修正をした場合は木の形になります。

## Roslyn での実装

### Green Tree

Roslyn では Green Tree は [src/Compilers/Core/Portable/Syntax/GreenNode.cs](https://github.com/dotnet/roslyn/blob/main/src/Compilers/Core/Portable/Syntax/GreenNode.cs) で定義されている GreenNode を継承しています。
説明したい部分を以下に抜き出しました。
```
internal abstract class GreenNode : IObjectWritable
{
    private readonly ushort _kind;
    private byte _slotCount;
    private int _fullWidth;

    internal abstract GreenNode? GetSlot(int index);

    ...
}
```
たしかに `_fullWidth` としてソースコード上の長さを保持しています。
また、親ノードを参照するメンバを持たないことから GreenNode の状態は子ノードにしか依存しないことが分かります。

### Red Tree

Red Tree は Roslyn では [src/Compilers/Core/Portable/Syntax/SyntaxNode.cs](https://github.com/dotnet/roslyn/blob/main/src/Compilers/Core/Portable/Syntax/SyntaxNode.cs) にて SyntaxNode という名前で定義されています。
説明したい部分を以下に抜き出しました。
```
public abstract partial class SyntaxNode
{
    private readonly SyntaxNode? _parent;

    internal GreenNode Green { get; }
    internal int Position { get; }
    internal int EndPosition => Position + Green.FullWidth;

    ...
}
```

こちらは Green Tree とは対称的に `Position` と `EndPosition` という名前でソースコード上の位置を持っていることが分かります。
Red Tree はプログラム全体の GreenNode が構築された後に作られるので、GreenNode の持つ長さを使うことでノードのソースコード上の位置を使うことができます。
また、`_parent` をフィールドに持っており、親や兄弟ノードへのアクセスという GreenNode では提供していなかった走査が可能になっています。

## Roslyn のコードを読みたい人向け

上記では Red Green Tree の説明のために Roslyn 上で具体的にどうなっているのかを省きました。
ここでは Roslyn のコードを読みたい人向けに Red Green Tree に関するソースコードがどのようになっているのかを説明します。

まず、Green Tree と Red Tree は別の木ですが、その構造は C# の文法と一致しているため共通部分が多くあります。
そのため Roslyn では [eng/generate-compiler-code.cmd](https://github.com/dotnet/roslyn/blob/main/eng/generate-compiler-code.cmd) を実行することで [src/Compilers/CSharp/Portable/Syntax/Syntax.xml](https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Syntax/Syntax.xml) という C# の文法を定義した XML から [src/Compilers/CSharp/Portable/Generated](https://github.com/dotnet/roslyn/tree/main/src/Compilers/CSharp/Portable/Generated) 以下に Green Tree と Red Tree の構造などを定義した C# コードを生成するようにしています。
生成されるファイルの内、今回取り上げたものに関係があるのは次の通りです。
- [src/Compilers/CSharp/Portable/Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/Syntax.xml.Internal.Generated.cs](https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/Syntax.xml.Internal.Generated.cs) : Green Tree のノードの定義とその Factory や Visitor が定義されています。Green Tree のノードはすべて GreenNode クラスを継承しています
- [src/Compilers/CSharp/Portable/Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/Syntax.xml.Syntax.Generated.cs](https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/Syntax.xml.Syntax.Generated.cs) : Red Tree のノードが定義されています。これらはすべて SyntaxNode クラス (Red Tree のノード) を継承しています
- [src/Compilers/CSharp/Portable/Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/Syntax.xml.Main.Generated.cs](https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/Syntax.xml.Main.Generated.cs) : Red Tree のノードに対する Factory や Visitor が定義されています

ちなみに [src/Compilers/CSharp/Portable/Generated/CSharp.Generated.g4](https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Generated/CSharp.Generated.g4) というファイルも生成されます。これは [ANTLR4](https://www.antlr.org/) というメジャーなパーザジェネレータ向けの文法定義ファイルですが、Roslyn はこれを使って構文解析をしているわけではなさそうです。
C# の構文解析は [src/Compilers/CSharp/Portable/Parser/LanguageParser.cs](https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs) でされていて、完全に手書きっぽい雰囲気があります。やはり究極までユーザフレンドリーなエラー報告をしようと思うと構文解析器手書きをすることになるんでしょうか。恐ろしいですね。

この後は https://github.com/KirillOsenkov/Bliki/wiki/Roslyn-Immutable-Trees で紹介されている Roslyn の最適化のいくつかがコード上のどこで行われているのかを見てましょう。

### 診断データや注釈データを静的メンバとして持つ

単純に考えると診断データや注釈データなどの構文木に対して付加されるデータは Green Tree の各ノードが持てばよさそうですが、Roslyn では GreenNode の静的メンバとして保持しています。(https://github.com/dotnet/roslyn/blob/main/src/Compilers/Core/Portable/Syntax/GreenNode.cs#L34)
ほとんどのノードは診断データなどを持たないため、このようにした方が GreenNode のサイズが小さくなりキャッシュヒット率が上がりパフォーマンスが良くなるそうです。

### 終端トークンを使いまわす

記号やキーワードのようなトークンについては予めトークンが作られていてそれを使いまわすようになっています。
ソースコード上では https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Syntax/InternalSyntax/SyntaxToken.cs#L146 で定義されており、そののちの static コンストラクタで初期化されています。(https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Syntax/InternalSyntax/SyntaxToken.cs#L151)
そして https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Syntax/InternalSyntax/SyntaxToken.cs#L113 にてトークンに先行/後続する空白などがあるかどうかを見て使いまわせるものは使いまわすようにしています。

### Red Tree のノード生成を遅延させる

ソースコードがパーズされた直後の Red Tree は Green Tree 側の CompilationUnitSyntax を指すだけの子要素を持たない木になります。(https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Syntax/CSharpSyntaxTree.cs#L507)
その後、この CompilationUnitSyntax のメンバを使用する時に子要素が構築されるようになっています。
例えばクラス定義の構文である ClassDeclarationSyntax (https://github.com/dotnet/roslyn/blob/main/src/Compilers/CSharp/Portable/Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/Syntax.xml.Syntax.Generated.cs#L10123) は以下のようになっています。
```
public sealed partial class ClassDeclarationSyntax : TypeDeclarationSyntax
{
    private SyntaxNode? attributeLists;
    private TypeParameterListSyntax? typeParameterList;
    private BaseListSyntax? baseList;
    private SyntaxNode? constraintClauses;
    private SyntaxNode? members;
    // ...
    public override SyntaxList<MemberDeclarationSyntax> Members => new SyntaxList<MemberDeclarationSyntax>(GetRed(ref this.members, 8));
}
```
上記のようになっており、例えばクラス定義のメンバを読みに行くタイミングで `GetRed(this.members, 8)` を呼んでいます。
`GetRed(ref this.members, 8)` の中では `this.members` が `null` かどうか判定し、`null` であればそれについて Red Node を作って返しています。

## 参考になりそうなもの
- Persistence, Facades and Roslyn's Red-Green Trees (https://learn.microsoft.com/ja-jp/archive/blogs/ericlippert/persistence-facades-and-roslyns-red-green-trees)
- https://github.com/rust-lang/rust-analyzer/blob/master/docs/dev/syntax.md
- https://willspeak.me/2021/11/24/red-green-syntax-trees-an-overview.html
- https://github.com/KirillOsenkov/Bliki/wiki/Roslyn-Immutable-Trees
- https://github.com/oilshell/oil/wiki/Lossless-Syntax-Tree-Pattern
