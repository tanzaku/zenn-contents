---
title: "Roslynで簡易コールグラフを作成するメモ"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "Roslyn"]
published: true
---

## 環境構築
https://docs.microsoft.com/ja-jp/dotnet/core/install/linux-ubuntu を参考にインストールします。

```bash
# パッケージ リポジトリを追加
wget https://packages.microsoft.com/config/ubuntu/20.10/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb

# SDK のインストール
sudo apt-get update; \
  sudo apt-get install -y apt-transport-https && \
  sudo apt-get update && \
  sudo apt-get install -y dotnet-sdk-5.0
```

C#プロジェクトを作成します。

```bash
dotnet new console
```

今回はRoslyn APIを使用して解析したいのでパッケージを追加します。
VisualStudioインストール済みのWindows環境であれば [MSBuildWorkspace](https://www.nuget.org/packages/Microsoft.CodeAnalysis.Workspaces.MSBuild/) を使用するとワークスペースの解析が簡単にできそうですが、筆者のUbuntu環境ではエラーになってしまうため使用しませんでした。

```bash
dotnet add package Microsoft.CodeAnalysis.CSharp --version 3.9.0
dotnet add package Microsoft.CodeAnalysis.CSharp.Workspaces --version 3.9.0
```

## 解析対象
今回は下記の2クラスを解析します。

```csharp
using System;
namespace Test.HelloWorld
{
    public class A
    {
        public static void F()
        {
            F();
            Console.WriteLine('X');
            new A().H("");
        }

        void H(string s)
        {
            Console.WriteLine(s);
        }
    }
}
```

```csharp
namespace Test.HelloWorld
{
    public class B
    {
        public static void F()
        {
            A.F();
        }
    }
}
```

## 解析処理

まずコールグラフのエッジ構築するのは下記のように行っています。
今回はSemanticModelを用いてCallerのメソッドとCalleeのメソッドのシンボルを取得し、それを元にエッジを構築します。

```csharp
using System;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;

namespace RoslynCSharpCallGraph.Walker
{
    public class CallGraphWalker : CSharpSyntaxWalker
    {
        private readonly SemanticModel semanticModel;

        public CallGraphWalker(SemanticModel semanticModel)
        {
            this.semanticModel = semanticModel;
        }

        private static string ExtractCallerID(SemanticModel semanticModel, SyntaxNode node)
        {
            while (node != null)
            {
                if (node is MethodDeclarationSyntax methodDeclarationNode)
                {
                    var methodSymbol = semanticModel.GetDeclaredSymbol(methodDeclarationNode);
                    return methodSymbol.ToDisplayString();
                }

                node = node.Parent;
            }

            throw new ArgumentException("");
        }

        public override void VisitInvocationExpression(InvocationExpressionSyntax node)
        {
            var calleeMethodSymbol = semanticModel.GetSymbolInfo(node).Symbol as IMethodSymbol;

            string callerMethodID = ExtractCallerID(semanticModel, node);
            string calleeMethodID = calleeMethodSymbol.ToDisplayString();
            Console.WriteLine("\t\"{0}\" -> \"{1}\";", callerMethodID, calleeMethodID);

            base.VisitInvocationExpression(node);
        }
    }
}
```

ワークスペースの構築は下記のようにしています。

```csharp
            // workspaceの構築
            var projectName = "TestProject";
            var assemblyName = "TestProject";
            var files = new List<string> { @"HelloWorld/A.cs", @"HelloWorld/B.cs" };

            var projectId = ProjectId.CreateNewId(projectName);

            var metadataReferences = new List<MetadataReference> {
                MetadataReference.CreateFromFile(typeof(object).Assembly.Location),
                MetadataReference.CreateFromFile(typeof(Console).Assembly.Location)
            };

            var documentInfos = new List<DocumentInfo>();
            foreach (var file in files)
            {
                var debugName = file;
                var documentId = DocumentId.CreateNewId(projectId, debugName);

                var source = File.ReadAllText(file);

                var version = VersionStamp.Create();
                var textAndVersion = TextAndVersion.Create(SourceText.From(source), version);
                var loader = TextLoader.From(textAndVersion);
                var filePath = file;

                var documentName = file;
                var documentInfo = DocumentInfo.Create(
                    documentId,
                    documentName,
                    sourceCodeKind: SourceCodeKind.Regular,
                    loader: loader,
                    filePath: filePath
                );

                documentInfos.Add(documentInfo);
            }

            var workspace = new AdhocWorkspace();

            var solution = workspace.CurrentSolution
                .AddProject(projectId, projectName, assemblyName, LanguageNames.CSharp)
                .AddMetadataReferences(projectId, metadataReferences)
                .AddDocuments(documentInfos.ToImmutableArray());
```

解析のメインの処理では、下記のようにDocumentを一つずつ解析して各ファイル毎にエッジを構築しています。
(今回は簡単のためにdot言語を標準出力に出力しています)

```csharp
            // 各Documentを解析してgraphvizを出力
            Console.WriteLine("digraph graph_name {");
            Console.WriteLine("\tgraph [ rankdir = LR ];");

            foreach (var documentInfo in documentInfos)
            {
                var document = solution.GetDocument(documentInfo.Id);
                var semanticModel = document.GetSemanticModelAsync().Result;
                var syntexTree = document.GetSyntaxTreeAsync().Result;
                var walker = new CallGraphWalker(semanticModel);
                walker.Visit(syntexTree.GetRoot());
            }

            Console.WriteLine("}");
```

上記で実装は終わりです。実行すると https://github.com/tanzaku/roslyn-c-sharp-callgraph/blob/main/graphviz.svg を生成するdot言語が出力されます。
コード全体は https://github.com/tanzaku/roslyn-c-sharp-callgraph で見ることができます。


## まとめ
Roslyn APIを用いることで、少ないコード量で静的解析することができました。
静的解析ができると、独自のLintルールを追加し品質向上につながるツールを作成するなど様々な応用ができそうですね。
