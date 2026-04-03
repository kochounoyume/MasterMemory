# T4テンプレートの仕組み調査 (T4 Template Mechanism Investigation)

## 概要 / Overview

`src/MasterMemory.SourceGenerator/GeneratorCore/` 内のT4テンプレートがどのように機能するかを説明する。

---

## ファイル構成 / File Structure

各テンプレートは以下の3ファイルセットで構成される：

```
DatabaseBuilderTemplate.tt    ← 人間が書く T4 テンプレート本体
DatabaseBuilderTemplate.cs    ← Visual Studio が .tt から自動生成するプリプロセッサ出力
                               （TransformText() を含む partial class）
Template.cs                   ← 手動で書くプロパティ定義 (partial class の片割れ)
```

---

## 仕組みの詳細 / Mechanism Details

### 1. `.tt` ファイル — T4テンプレート本体

T4 の構文は 3 種類に分類される：

#### ① `<#@ directive #>` — ディレクティブ（テンプレート設定）

```
<#@ template debug="false" hostspecific="false" linePragmas="false" language="C#" #>
<#@ assembly name="System.Core" #>
<#@ import namespace="System.Linq" #>
<#@ import namespace="System.Text" #>
<#@ import namespace="System.Collections.Generic" #>
```

ディレクティブはテンプレートエンジン自体の設定を行う。出力テキストには**何も書き出さない**。

| ディレクティブ | 効果 |
|---|---|
| `<#@ template ... #>` | 使用言語・デバッグ設定等を指定 |
| `<#@ assembly name="..." #>` | 通常の T4 では参照アセンブリを追加するが、**プリプロセッサ方式では .csproj のプロジェクト参照が使われるため実質的に無視される** |
| `<#@ import namespace="..." #>` | 生成される `.cs` プリプロセッサファイル内に `using` ディレクティブを追加する（出力テキストへの書き出しではない） |

> **重要：** `<#@ import namespace="System.Linq" #>` は `TransformText()` が返す**出力テキスト**に `using System.Linq;` を書き出すのではなく、`TransformText()` メソッド自体を含む**生成クラスファイル**（`DatabaseBuilderTemplate.cs`）に `using System.Linq;` を追加する。これによりテンプレートの C# コードブロック内で `System.Linq` の型を使用できるようになる。

#### ② `<#= expr #>` — 式ブロック（出力）

```
<#= Using #>
namespace <#= Namespace #>
```

式の評価結果を出力テキストに文字列として書き出す。

#### ③ `<# code #>` — コードブロック（制御）

```
<# foreach(var item in GenerationContexts) { #>
    public ... Append(IEnumerable<<#= item.ClassName #>> dataSource) { ... }
<# } #>
```

C# コードを直接実行する（`foreach`、`if` 等）。出力テキストへの書き出しは行わない。

#### ④ 静的テキスト

上記以外のテキストはそのまま出力文字列になる。

---

### 2. `.cs` ファイル — T4プリプロセッサ出力

Visual Studio の `TextTemplatingFilePreprocessor` が `.tt` ファイルをビルド時（Design Time）に解析して生成するファイル。

**`.tt` → `.cs` 変換の対応関係：**

| `.tt` の記述 | `.cs` への変換結果 |
|---|---|
| `<#@ import namespace="System.Linq" #>` | クラスファイル冒頭に `using System.Linq;` が追加される（`TransformText()` の出力テキストではない） |
| `<#= Using #>` | `TransformText()` 内: `this.Write(this.ToStringHelper.ToStringWithCulture(Using));` |
| `namespace <#= Namespace #>` | `TransformText()` 内: `this.Write("namespace ");` + `this.Write(...Namespace...);` |
| `<# foreach(...) { #>` | `TransformText()` 内: `foreach(...) {` （C#コードとしてインライン展開）|
| 静的テキスト `"hello"` | `TransformText()` 内: `this.Write("hello");` |

生成される `TransformText()` は最終的に `GenerationEnvironment.ToString()` を返す（StringBuilder の内容）。

**注意：** `.cs` ファイルは **コミットされた生成済みファイル** であり、Visual Studio環境外（CI等）でも動作するよう意図的にリポジトリに含まれている。

### 3. `Template.cs` — partial class によるプロパティ定義

`.cs` ファイルは以下のように `partial class` として宣言される：

```csharp
// DatabaseBuilderTemplate.cs (自動生成)
public partial class DatabaseBuilderTemplate : DatabaseBuilderTemplateBase
{
    public virtual string TransformText() { ... }
}
```

`Template.cs` はその **片割れ** となる `partial class` を手動で定義し、プロパティを追加する：

```csharp
// Template.cs (手動)
public partial class DatabaseBuilderTemplate
{
    public string Namespace { get; set; }
    public string Using { get; set; }
    public string PrefixClassName { get; set; }
    public GenerationContext[] GenerationContexts { get; set; }
    public string ClassName => PrefixClassName + "DatabaseBuilder";
}
```

コンパイル時にこの2つの `partial class` が合体し、`TransformText()` の中で `Namespace` や `Using` 等のプロパティが参照できるようになる。

### 4. `MasterMemoryGenerator.cs` — プロパティへの値の注入

Roslyn ソースジェネレーターが実行時にテンプレートのプロパティをセットする：

```csharp
// Namespace の取得元：
// 1. [MasterMemoryGeneratorOptions(Namespace = "...")] アトリビュート（ユーザー指定）
// 2. build_property.RootNamespace（プロジェクトの RootNamespace MSBuild プロパティ）
// 3. フォールバック: "MasterMemory"
var usingNamespace = generatorOptions.Namespace ?? defaultNamespace ?? "MasterMemory";

// Using の取得元：
// ユーザーコードの各 [MemoryTable] クラスの using 宣言を収集 + Tables名前空間を追加
var usingStrings = string.Join(Environment.NewLine,
    memoryTables.SelectMany(x => x.UsingStrings)
                .Distinct()
                .OrderBy(x => x, StringComparer.Ordinal));

var builderTemplate = new DatabaseBuilderTemplate();
builderTemplate.Namespace = usingNamespace;    // ← ここでプロパティに値をセット
builderTemplate.Using = usingStrings + ...;
builderTemplate.GenerationContexts = memoryTables.ToArray();

// TransformText() を呼ぶと生成済みのC#コード文字列が返る
context.AddSource("MasterMemory.DatabaseBuilder.g.cs", builderTemplate.TransformText());
```

---

## データフロー / Data Flow

```
[MemoryTable] クラスのソースコード
        ↓ Roslyn構文解析 (CodeGenerator.CreateGenerationContext)
GenerationContext[] (クラス名・主キー・インデックス等の情報)
        ↓
MasterMemoryGenerator.EmitMemoryTable()
  ├── Namespace = RootNamespace or [MasterMemoryGeneratorOptions].Namespace
  ├── Using     = ユーザーコードの using 宣言 + "using {Namespace}.Tables;"
  └── PrefixClassName = [MasterMemoryGeneratorOptions].PrefixClassName or ""
        ↓
DatabaseBuilderTemplate.TransformText()  ← .tt の変換結果である .cs の TransformText()
        ↓
生成されたC#コード文字列 (例: "DatabaseBuilder.g.cs" の内容)
```

---

## `MetaMemoryDatabaseTemplate` について

`Template.cs` には `MetaMemoryDatabaseTemplate` の `partial class` 定義が存在するが：

- 対応する `.tt` ファイルが**存在しない**
- 対応する `.cs` (プリプロセッサ出力) が**存在しない**
- `MasterMemoryGenerator.cs` からも**使用されていない**

現状では **未実装のプレースホルダー** となっている。`MemoryDatabaseTemplate.tt` で生成される `MemoryDatabase` クラスが `GetMetaDatabase()` を内包しているため、独立した `MetaMemoryDatabase` クラスの実装は行われていない。

---

## .csproj の設定との関係

`.csproj` の以下の設定が T4 のビルドとの関係を定義する：

```xml
<!-- .tt を TextTemplatingFilePreprocessor で処理することを宣言 -->
<None Update="GeneratorCore\DatabaseBuilderTemplate.tt">
    <Generator>TextTemplatingFilePreprocessor</Generator>
    <LastGenOutput>DatabaseBuilderTemplate.cs</LastGenOutput>
</None>

<!-- 生成済み .cs をコンパイル対象に含める（Visual Studio のデザイン時生成との連動） -->
<Compile Update="GeneratorCore\DatabaseBuilderTemplate.cs">
    <DesignTime>True</DesignTime>
    <AutoGen>True</AutoGen>
    <DependentUpon>DatabaseBuilderTemplate.tt</DependentUpon>
</Compile>
```

この設定により Visual Studio のソリューションエクスプローラーでは `.cs` が `.tt` の子項目として表示される。CI/CDビルドでは `.tt` の処理は行われず、コミット済みの `.cs` がそのままコンパイルされる。
