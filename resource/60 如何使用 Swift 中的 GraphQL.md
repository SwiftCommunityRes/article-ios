## 前言

我一直在分享关于类型安全和在 Swift 中构建健壮 API 的更多内容。今天，我想继续探讨类型安全的话题，介绍 GraphQL。GraphQL 是一种用于 API 的查询语言。本周，我们将讨论 GraphQL 的好处，并学习如何在 Swift 中使用它。

## 基础知识

首先介绍一下 GraphQL。GraphQL 是一种用于 API 的查询语言。通常，后端开发人员或网络服务会为你提供一个模式文件和一个 GraphQL 端点。模式文件包含所有你可以使用该端点进行的类型和查询。让我们来看一个模式文件的例子。

```graphql
schema {
  query: Query
  mutation: Mutation
}

type Query {
  film(id: ID, filmID: ID): Film
  allFilms(after: String, first: Int, before: String, last: Int): FilmsConnection
  """更多代码"""
}
```

模式文件应包含 `Query` 和 `Mutation` 类型。这些类型定义了当前 GraphQL 端点支持的所有查询和变更操作。模式文件还描述了你可以在查询中使用的所有类型的列表。

```graphql
type Film implements Node {
  title: String!
  episodeID: Int
  openingCrawl: String
  director: String!
}
```

GraphQL 是一种强类型语言。GraphQL 自定义类型中的每个字段都必须声明其类型。默认情况下，每个字段都可以为 nil。带有感叹号的字段不能为 nil。

我使用星球大战 API 来向你展示本文中的示例。让我们继续进行一些查询。你可以通过 GraphiQL 应用轻松玩转 GraphQL API，使用以下端点。

```swift
query AllFilms {
  allFilms {
    films {
      title
    }
  }
}
```

**响应：**

```json
{
  "data": {
    "allFilms": {
      "films": [
        {
          "title": "A New Hope"
        },
        {
          "title": "The Empire Strikes Back"
        },
        {
          "title": "Return of the Jedi"
        },
        {
          "title": "The Phantom Menace"
        },
        {
          "title": "Attack of the Clones"
        },
        {
          "title": "Revenge of the Sith"
        }
      ]
    }
  }
}
```

如你所见，我们使用模式文件中的数据类型构建我们的查询。我喜欢GraphQL的一点是响应格式。请求格式直接映射到响应格式。你可以在请求中添加更多字段，响应也会包含它们。

```swift
query AllFilms {
  allFilms {
    films {
      title
      director
    }
  }
}
```

**响应：**

```json
{
  "data": {
    "allFilms": {
      "films": [
        {
          "title": "A New Hope",
          "director": "George Lucas"
        },
        {
          "title": "The Empire Strikes Back",
          "director": "Irvin Kershner"
        },
        {
          "title": "Return of the Jedi",
          "director": "Richard Marquand"
        },
        {
          "title": "The Phantom Menace",
          "director": "George Lucas"
        },
        {
          "title": "Attack of the Clones",
          "director": "George Lucas"
        },
        {
          "title": "Revenge of the Sith",
          "director": "George Lucas"
        }
      ]
    }
  }
}
```

使用 GraphQL，我们只获取我们请求的数据，绝不会多余。

## ApolloGraphQL

ApolloGraphQL 是一个很棒的框架，它可以让你轻松进行 GraphQL 查询和变更。ApolloGraphQL iOS 框架负责缓存和代码生成。ApolloGraphQL 为你在项目中定义的查询和变更生成 Swift 类型。它通过自动生成所有样板代码来节省你的时间。

以下是将 `ApolloGraphQL` 设置到项目中的一些步骤：

1. 你应该使用SPM或其他包管理器将 `ApolloGraphQL` 嵌入到你的项目中。
2. 在编译源代码部分上方的构建阶段添加运行脚本。这个脚本下载模式并为你的查询生成 Swift 类型。你可以在这个脚本中轻松更改 GraphQL 端点以连接到你的 GraphQL 后端。

我们已准备好使用 `ApolloGraphQL` 的项目。现在我们可以向项目添加第一个查询。我们应该在项目中创建一个带有 `.graphql` 扩展名的文件，并将这些行放入文件中。

```graphql
query AllFilms {
  allFilms {
    films {
      title
      director
    }
  }
}
```

让我们现在构建项目。`ApolloGraphQL` 生成一个 `API.swift` 文件，你应该将其添加到项目中。所有需要的类型都在这里，可以非常类型安全地进行 `GraphQL` 查询。每个请求类型都定义了其响应类型。`ApolloGraphQL` 生成了 `AllFilmsQuery` 和 Data 类型，描述了请求和响应。现在我们可以使用生成的代码进行 GraphQL 请求。

```swift
let url = URL(string: "https://swapi-graphql.netlify.app/.netlify/functions/index")!
let client = ApolloClient(url: url)

client.fetch(query: AllFilmsQuery()) { result in
    switch result {
    case .success(let response):
        print(response.data?.allFilms?.films ?? [])
    case .failure(let error):
        print(error)
    }
}
```

## 结论

GraphQL 为 API 开发带来了诸多优势，尤其是在类型安全和数据查询方面。通过定义明确的模式文件，GraphQL 确保了请求和响应的一致性，使得开发者能够精准获取所需数据，避免多余信息的传输。此外，GraphQL 强类型的特性进一步提升了代码的可靠性和可维护性。

在 Swift 中，ApolloGraphQL 框架极大地简化了 GraphQL 查询和变更的实现过程，自动生成的 Swift 类型和缓存机制不仅提高了开发效率，还减少了样板代码的编写。总之，GraphQL 是一种高效、灵活且类型安全的API解决方案，适用于构建现代化应用程序。尽管 GraphQL 也有其挑战，但其带来的优势使其成为 REST API 的有力竞争者。通过不断探索和优化，GraphQL 将在更多项目中得到广泛应用。