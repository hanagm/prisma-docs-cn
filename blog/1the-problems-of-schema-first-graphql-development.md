# “Schema优先”GraphQL服务器开发的问题

> 翻译:[hanagm](https://github.com/hanagm)
>
> [原文链接](https://www.prisma.io/blog/the-problems-of-schema-first-graphql-development-x1mn4cb0tyl3/)



GraphQL服务器开发的工具在过去两年中爆炸式增长。我们认为对大多数工具的需求来自流行的Schema优先方法 - 并且可以通过替代方案解决：代码优先。

![img](https://www.prisma.io/static/posts/the-problems-of-schema-first-graphql-development.png)



本文是由三部分组成的系列文章的*第一*部分：

- 第1部分：**“Schema优先”GraphQL服务器开发的问题**
- 第2部分：[介绍GraphQL Nexus：代码优先的GraphQL服务器开发](./2introducing-graphql-nexus-code-first-graphql-server.md)
- 第3部分：[将GraphQL Nexus与数据库一起使用](./3using-graphql-nexus-with-a-database.md)

**在Twitter上关注我们，**以便在即将发布的文章时收到通知。





------

## 概述：从Schema优先到代码优先

本文概述了GraphQL服务器开发空间的当前状态。以下是所涵盖内容的快速概述：

1. 本文中“schema优先”的含义是什么？
2. GraphQL服务器开发的演变
3. 分析SDL优先发展的问题
4. 结论：SDL-first可能有用，但需要大量工具
5. 代码优先：GraphQL服务器开发的一种语言惯用方式

虽然本文主要提供JavaScript生态系统的示例，但其中大部分也适用于其他语言生态系统中的GraphQL服务器开发。





------

### 本文中“schema优先”的含义是什么？

schema优先*这个术语非常模糊，总的来说传达了一个非常积极的想法：**使schema设计成为开发过程中的优先事项。**

在实现schema之前考虑模式（以及API）通常会产生更好的API设计。如果schema设计不尽如人意，那么最终会遇到API的风险，这是后端实现方式的结果，忽略了业务领域的原语和API使用者的需求。

在本文中，我们将讨论开发过程的缺点，其中GraphQL架构首先在SDL中*手动*定义，之后实现解析器。在这种方法中，[SDL](https://www.prisma.io/blog/graphql-sdl-schema-definition-language-6755bcb9ce51)是API *的真实来源*。**为了阐明schema优先设计与这种特定实现方法之间的区别，我们将从此处首先将其称为SDL。**

相比之下，**代码优先**（有时也称为*解析器优先*）是一个以*编程*方式实现GraphQL架构的过程，而架构的SDL版本是*生成的工件*。使用代码优先，您仍然可以非常注意前期schema设计！

------

### GraphQL服务器开发的演变

![img](https://i.imgur.com/jFszNXY.png)

#### 阶段1：早期与 `graphql-js`

当GraphQL于2015年发布时，工具生态系统很少。在JavaScript中只有官方[规范](https://facebook.github.io/graphql/)及其参考实现：[`graphql-js`](https://github.com/graphql/graphql-js)。直到今天，`graphql-js`也是用于最流行的GraphQL服务器，如`apollo-server`，`express-graphql`和`graphql-yoga`。

使用`graphql-js`构建GraphQL服务器时，[GraphQL架构](https://www.prisma.io/blog/graphql-server-basics-the-schema-ac5e2950214e/)被定义为纯JavaScript对象：

***Hello Wrold***

```javascript
const { 
  GraphQLSchema, 
  GraphQLObjectType, 
  GraphQLString 
} = require('graphql')

const schema = new GraphQLSchema({
  query: new GraphQLObjectType({
    name: 'Query',
    fields: {
      hello: {
        type: GraphQLString,
        args: {
          name: { type: GraphQLString }
        },
        resolve: (_, args) => `Hello ${args.name || 'World!'}`
      }
    }
  })
})
```

***User Example***

```javascript
const { 
  GraphQLSchema, 
  GraphQLObjectType, 
  GraphQLString,
  GraphQLID
} = require('graphql')

const UserType = new GraphQLObjectType({
  name: 'User',
  fields: {
    id: { type: GraphQLID },
    name: { type: GraphQLString }
  }
})

const schema = new GraphQLSchema({
  query: new GraphQLObjectType({
    name: 'Query',
    fields: {
      user: {
        type: UserType,
        args: {
          id: { type: GraphQLID }
        },
        resolve: (_, { id }) => {
          return fetchUserById(id) // fetch the user, e.g. from a database
        }
      }
    }
  }),
  mutation: new GraphQLObjectType({
    name: 'Mutation',
    fields: {
      createUser: {
        type: UserType,
        args: {
          name: { type: GraphQLString }
        },
        resolve: (_, { name }) => {
          return createUser(name) // create a user, e.g. in a database
        }
      }
    }
  }),
})
```

从这些示例中可以看出，用于创建GraphQL模式的API `graphql-js`非常详细。模式的SDL表示更简洁，更容易掌握：

***Hello World (SDL)***

```graphql
type Query {
  hello(name: String): String
}
```

***User Example (SDL)***

```g
type User {
  id: ID
  name: String
}
type Query {
  user(id: ID): User
}
type Mutation {
  createUser(name: String): User
}
```

> 详细了解如何构建GraphQL模式与`graphql-js`在[此](https://www.prisma.io/blog/graphql-server-basics-the-schema-ac5e2950214e/)文章。
>
>

#### 阶段2：Schema优先推广 `graphql-tools`

为了简化开发并提高对实际API定义的可见性，Apollo [`graphql-tools`](https://github.com/apollographql/graphql-tools)于2016年3月开始构建库（[这](https://github.com/apollographql/graphql-tools/commit/fe17783a6a166278195d172beb5670b045fb300d)是第一次提交）。

目标是将模式*定义*与实际*实现*分开，这导致了当前流行的*模式驱动*或*模式优先* / *SDL优先*开发过程：

1. 在GraphQL SDL中手动编写GraphQL架构定义
2. 实现所需的解析器功能

通过这种方法，上面的示例现在看起来像这样

***Hello World***

```javascript
const { makeExecutableSchema } = require('graphql-tools')

const typeDefs = `
type Query {
  hello(name: String): String
}
`

const resolvers = {
  Query: {
    hello: (_, args) => `Hello ${args.name || 'World!'}`
  }
}

const schema = makeExecutableSchema({
  typeDefs,
  resolvers
})
```

***User Example***

```javascript
const { makeExecutableSchema } = require('graphql-tools')

const typeDefs = `
type User {
  id: ID
  name: String
}
type Query {
  user(id: ID): User
}
type Mutation {
  createUser(name: String): User
}
`

const resolvers = {
  Query: {
    user: (_, args) => fetchUserById(args.id) // fetch the user, e.g. from a database
  },
  Mutation: {
    createUser: (_, args) => createUser(args.name) // create a user, e.g. in a database
  }
}

const schema = makeExecutableSchema({
  typeDefs,
  resolvers
})
```

这些代码片段与上面使用的代码完全等效`graphql-js`，除了它们更易读和更容易理解。

可读性不是SDL优先的唯一优势：

- 这种方法**易于理解**，**非常适合快速构建**
- 由于每个新的API操作首先需要在模式定义中体现出来，因此**GraphQL模式设计不是经过深思熟虑的**
- 架构定义可以用作**API文档**
- 模式定义可以作为**前端和后端团队之间**的**通信工具** - 前端开发人员获得授权并更多地参与API设计
- 模式定义可以**快速**模拟**API**

#### 阶段3：开发新工具以“先修复”SDL

虽然SDL-first具有许多优势，但过去两年表明将其扩展到更大的项目是一项挑战。在更复杂的环境中会出现许多问题（我们将在下一节中详细讨论这些问题）。

这些问题本身确实是可以解决的 - *实际*问题是解决这些问题需要使用（并*学习*）许多其他工具。在过去的两年中，已经发布了大量的工具，这些工具正在尝试改进围绕SDL第一次开发的工作流程：从编辑器插件到CLI，再到语言库。

学习，管理和集成所有这些工具的开销减慢了开发人员的速度，并且很难跟上GraphQL生态系统的步伐。





------

### 分析SDL优先发展的问题

现在让我们深入探讨SDL-first开发的问题领域。请注意，大多数这些问题特别适用于当前的JavaScript生态系统。



#### 问题1：模式定义和解析器之间的不一致

使用SDL优先，模式定义*必须*与解析器实现的确切结构相匹配。这意味着开发人员需要确保模式定义始终与解析器同步！

虽然这对于小型模式来说已经是一个挑战，但是随着模式增长到数百或数千行，实际上变得不可能（作为参考，[GitHub GraphQL schema](https://gist.github.com/nikolasburk/15f30bb3e19e5bf8329ff52787fa72b5)有超过10k行）。

**工具/解决方案：**有一些工具可以帮助保持模式定义和解析器同步。例如，通过使用像[`graphqlgen`](https://github.com/prisma/graphqlgen)或等库来生成代码[`graphql-code-generator`](https://github.com/dotansimha/graphql-code-generator)。

> 不要以为这是一个问题？从`graphqlgen` [文章中](https://www.prisma.io/blog/graphqlgen-fj3s0ssc1jsx)获取我们的测验。



#### 问题2：GraphQL模式的模块化

编写大型GraphQL模式时，通常不希望所有GraphQL类型定义都驻留在同一文件中。相反，您希望将它们分成更小的部分（例如，根据*功能*或*产品*）。

![image-20190308140805155](./assets/image-20190308140805155.png)



**工具/解决方案：**像[`graphql-import`](https://github.com/prisma/graphql-import)或最近的[`graphql-modules`](https://graphql-modules.com/)这样的工具可以帮助解决。`graphql-import`使用自定义导入语法编写为SDL注释。`graphql-modules`是一个工具集，可以帮助解决*模式分离*，*解析程序组合*以及GraphQL服务器的*可伸缩结构*的实现。

#### 问题3：模式定义中的冗余（代码重用）

另一个问题是如何*重用* SDL定义。此问题的一个常见示例是中继式连接。虽然提供了一种实现分页的强大方法，但它们需要*大量*的样板和重复的代码。

目前没有工具可以帮助解决这个问题。开发人员可以编写自定义工具来减少重复代码的需要，但此问题目前缺乏通用解决方案。

#### 问题4：IDE支持和开发人员体验

GraphQL架构基于强类型系统，在开发过程中可以带来巨大的好处，因为它允许对代码进行静态分析。遗憾的是，SDL通常在程序中表示为纯*字符串*，这意味着工具无法识别其中的任何结构。

接下来的问题是如何在编辑器工作流中利用GraphQL类型，以便从SDL代码的自动完成和构建时错误检查等功能中受益。

**工具/解决方案：**该[`graphql-tag`](https://github.com/apollographql/graphql-tag)库暴露`gql`，轮流一个GraphQL字符串转换成一个AST功能，因此能够使静态分析和从以下的特征。除此之外，还有各种编辑器插件，例如用于VS Code 的[GraphQL](https://marketplace.visualstudio.com/items?itemName=Prisma.vscode-graphql)或[Apollo GraphQL](https://marketplace.visualstudio.com/items?itemName=apollographql.vscode-apollo)插件。



#### 问题5：编写GraphQL模式

模块化架构的想法也引出了另一个问题：如何*撰写*了许多现有的（分布式）架构成一个模式。

**工具/解决方案：**最常用的模式组合方法是[模式拼接](https://www.apollographql.com/docs/graphql-tools/schema-stitching.html)，它也是上述`graphql-tools`库的一部分。为了更好地控制模式的组成方式，您还可以直接使用[模式委派](https://www.prisma.io/blog/graphql-schema-stitching-explained-schema-delegation-4c6caf468405)（它是模式拼接的*子集*）。

### 结论：SDL-first可能*有用*，但需要大量工具

在探索了解决它们的问题领域和各种工具之后，似乎SDL第一次开发最终*可以*发挥作用 - 而且它还要求开发人员学习和使用大量其他工具。

![image-20190308141238294](./assets/image-20190308141238294.png)



#### 解决方法，解决方法，解决方法，......

在Prisma，我们在推动GraphQL生态系统发展方面发挥了重要作用。许多提到的工具都是由我们的工程师和社区成员构建的。

![img](https://imgur.com/OLITh8y.png)

经过几个月的开发和与GraphQL社区的密切互动，我们逐渐意识到我们只是在修复症状。这就像打一场[九头蛇](https://en.wikipedia.org/wiki/Lernaean_Hydra) - 解决一个问题导致一些新的问题。



#### 生态系统锁定：购买整个工具链

我们非常感谢Apollo的朋友们的工作，他们不断致力于改进围绕SDL开发的开发工作流程。

另一个以SDL优先构建GraphQL服务器的流行示例是[AWS AppSync](https://aws-amplify.github.io/docs/js/api#updating-your-graphql-schema)。它与Apollo模型略有不同，因为解析器（通常）不是以编程方式实现，而是从模式定义自动生成。

虽然社区从这么多工具中获益匪浅，但是当开发人员需要对某个组织的工具链进行全面赌注时，存在生态系统锁定的风险。在*真正的*解决方案可能会烤许多SDL-第一意见到GraphQL核心本身-这是不太可能在可预见的将来发生。



#### SDL首先忽略了编程语言的各个特征

SDL优先的另一个问题是，无论使用哪种编程语言，它都会通过强加类似的原则来忽略编程语言的各个特征。

代码优先方法在其他语言中运行得非常好：Scala库[`sangria-graphql`](https://github.com/sangria-graphql/sangria)利用Scala强大的类型系统来优雅地构建GraphQL模式，[`graphlq-ruby`](https://github.com/rmosolgo/graphql-ruby)使用Ruby语言的许多令人敬畏的DSL功能。



------



### 代码优先：GraphQL服务器开发的一种语言惯用方式



#### 您需要的唯一工具是您的编程语言

大多数SDL优先问题来自于我们需要*将手动编写的SDL模式映射到编程语言*。此映射是导致需要其他工具的原因。如果我们遵循SDL优先路径，则需要为*每个*语言生态系统重新创建所需的工具，并且*每个*语言生态系统的*外观也*不同。

我们应该努力寻求一种更简单的开发模型，而不是使用更多工具来增加GraphQL服务器开发的复杂性。理想情况下，它允许开发人员利用他们已经使用的编程语言 - 这就是*代码优先*的想法。



#### 什么是代码优先？

还记得定义架构的最初示例`graphql-js`吗？这是代码优先意味着什么的*本质*。没有手动维护的模式定义版本，而是从实现模式的代码*生成* SDL 。



虽然API `graphql-js`非常冗长，但是其他语言中有许多流行的框架基于代码优先方法工作，例如已经提到的，以及Python或Elixir。[`graphlq-ruby`](https://github.com/rmosolgo/graphql-ruby) [`sangria-graphql`](https://github.com/sangria-graphql/sangria)[`graphene`](https://github.com/graphql-python/graphene)[`absinthe-graphql`](https://github.com/absinthe-graphql/absinthe)



![image-20190308141926516](./assets/image-20190308141926516.png)





#### 代码优先实践

虽然本文主要是关于首先理解SDL的问题，但是对于构建具有代码优先框架的GraphQL架构的内容来说，这是一个小小的片段：

Hello World

```javascript
const Query = objectType('Query', t => {
  t.string('hello', {
    args: { name: stringArg() },
    resolve: (_, { name }) => `Hello ${name || `World`}!`,
  })
})

const schema = makeSchema({
  types: [Query],
  outputs: {
    schema: './schema.graphql'),
    typegen: './typegen.ts',
  },
})
```

User Example

```javascript
const User = objectType('User', t => {
  t.id('id')
  t.string('name', { nullable: true })
})

const Query = objectType('Query', t => {
  t.field('user', 'User', {
    args: { id: idArg() },
    resolve: (_, { id }) => fetchUserById(id) // fetch the user, e.g. from a database
  })
})

const Mutation = objectType('Query', t => {
  t.field('createUser', 'User', {
    args: { name: stringArg() },
    resolve: (_, { name }) => createUser(name) // create a user, e.g. in a database
  })
})

const schema = makeSchema({
  types: [User, Query, Mutation],
  outputs: {
    schema: './schema.graphql'),
    typegen: './typegen.ts',
  },
})
```

通过这种方法，您可以直接在TypeScript / JavaScript中定义GraphQL类型。通过正确的设置并且由于[智能代码完成](https://en.wikipedia.org/wiki/Intelligent_code_completion)，您的编辑人员将能够在您定义时建议可用的GraphQL类型，字段和参数。

典型的编辑器工作流程包括在后台运行的开发服务器，可在保存文件时重新生成类型。

一旦定义了所有GraphQL类型，就会将它们传递给函数以创建[`GraphQLSchema`](https://graphql.org/graphql-js/type/#graphqlschema)可在GraphQL服务器中使用的实例。通过指定`ouputs`，您可以定义生成的SDL和打字所在的位置。

本系列文章的下一部分将更详细地讨论代码优先开发。

#### 首先获得SDL的好处，而无需使用所有工具



之前我们列举了SDL优先开发的好处。事实上，在使用代码优先方法时，没有必要在大多数方面妥协。

​	**使用GraphQL架构作为前端和后端团队的关键通信工具的最重要的好处仍然存在。**

以GitHub GraphQL API为例：GitHub使用Ruby和代码优先的方法来实现他们的API。SDL架构定义是基于实现API的代码生成的。但是，架构定义仍会检入版本控制。这使得在开发过程中跟踪API的更改非常容易，并且可以改善各个团队之间的通信。

API文档或授权前端开发人员等其他好处也不会因代码优先方法而丢失。



### 代码优先的框架，很快就会进入你的IDE

这篇文章相当理论化，并没有包含太多代码 - 我们仍然希望能引起您对代码优先开发的兴趣。要查看更多实际示例并了解有关代码优先开发体验的更多信息，请继续关注并在接下来的几天内关注[Prisma Twitter帐户](https://www.twitter.com/prisma) 👀



> 你觉得这篇文章怎么样？加入**[Prisma Slack](https://slack.prisma.io/)**，与其他GraphQL爱好者一起讨论SDL优先和代码优先开发。
>
>