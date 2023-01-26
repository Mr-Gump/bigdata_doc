# 图表

图表有助于表达不同组件之间的复杂关系和联系，并且是对项目文档的一个很好的补充。Material for MkDocs 集成了 [Mermaid.js]，这是一种非常流行且灵活的绘图解决方案。

  [Mermaid.js]: https://mermaid-js.github.io/mermaid/

## 配置

请将下面几行添加到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
```

不需要进一步配置。相对于自定义集成有如下优势:

- [x] 使用即时加载，无需任何额外的工作
- [x] 图表自动使用 `mkdocs.yml`[^1] 中定义的字体和颜色
- [x] 同时支持浅色和深色配色方案——_在本页试试吧_!

  [^1]:
    虽然所有的 [Mermaid.js] 特性都是开箱即用的，但是 Material for MkDocs 目前只会调整流程图、序列图、类图、状态图和实体关系图的字体和颜色。请参阅其他图表类型部分，了解为什么目前没有为所有图表实现此功能的更多信息。

  [Diagrams support]: https://github.com/squidfunk/mkdocs-material/releases/tag/8.2.0


## 使用

### 使用流程图

[流程图] 是表示工作流或流程的图。步骤被呈现为各种类型的节点，并由边连接，描述了步骤的必要顺序:

```` markdown title="流程图"
``` mermaid
graph LR
  A[Start] --> B{Error?};
  B -->|Yes| C[Hmm...];
  C --> D[Debug];
  D --> B;
  B ---->|No| E[Yay!];
```
````

<div class="result" markdown>

``` mermaid
graph LR
  A[Start] --> B{Error?};
  B -->|Yes| C[Hmm...];
  C --> D[Debug];
  D --> B;
  B ---->|No| E[Yay!];
```

</div>

  [流程图]: https://mermaid-js.github.io/mermaid/#/flowchart

### 使用序列图

[序列图] 将特定场景描述为多个对象或参与者之间的顺序交互，包括在这些参与者之间交换的消息:

```` markdown title="序列图"
``` mermaid
sequenceDiagram
  autonumber
  Alice->>John: Hello John, how are you?
  loop Healthcheck
      John->>John: Fight against hypochondria
  end
  Note right of John: Rational thoughts!
  John-->>Alice: Great!
  John->>Bob: How about you?
  Bob-->>John: Jolly good!
```
````

<div class="result" markdown>

``` mermaid
sequenceDiagram
  autonumber
  Alice->>John: Hello John, how are you?
  loop Healthcheck
      John->>John: Fight against hypochondria
  end
  Note right of John: Rational thoughts!
  John-->>Alice: Great!
  John->>Bob: How about you?
  Bob-->>John: Jolly good!
```

</div>

  [序列图]: https://mermaid-js.github.io/mermaid/#/sequenceDiagram

### 使用状态图

[状态图] 是描述系统行为、将其分解为有限数量的状态以及这些状态之间的转换的绝佳工具:

```` markdown title="状态图"
``` mermaid
stateDiagram-v2
  state fork_state <<fork>>
    [*] --> fork_state
    fork_state --> State2
    fork_state --> State3

    state join_state <<join>>
    State2 --> join_state
    State3 --> join_state
    join_state --> State4
    State4 --> [*]
```
````

<div class="result" markdown>

``` mermaid
stateDiagram-v2
  state fork_state <<fork>>
    [*] --> fork_state
    fork_state --> State2
    fork_state --> State3

    state join_state <<join>>
    State2 --> join_state
    State3 --> join_state
    join_state --> State4
    State4 --> [*]
```

</div>

  [状态图]: https://mermaid-js.github.io/mermaid/#/stateDiagram

### 使用类图

[类图] 是面向对象编程的核心，通过将实体建模为类及其之间的关系来描述系统的结构:

```` markdown title="类图"
``` mermaid
classDiagram
  Person <|-- Student
  Person <|-- Professor
  Person : +String name
  Person : +String phoneNumber
  Person : +String emailAddress
  Person: +purchaseParkingPass()
  Address "1" <-- "0..1" Person:lives at
  class Student{
    +int studentNumber
    +int averageMark
    +isEligibleToEnrol()
    +getSeminarsTaken()
  }
  class Professor{
    +int salary
  }
  class Address{
    +String street
    +String city
    +String state
    +int postalCode
    +String country
    -validate()
    +outputAsLabel()  
  }
```
````

<div class="result" markdown>

``` mermaid
classDiagram
  Person <|-- Student
  Person <|-- Professor
  Person : +String name
  Person : +String phoneNumber
  Person : +String emailAddress
  Person: +purchaseParkingPass()
  Address "1" <-- "0..1" Person:lives at
  class Student{
    +int studentNumber
    +int averageMark
    +isEligibleToEnrol()
    +getSeminarsTaken()
  }
  class Professor{
    +int salary
  }
  class Address{
    +String street
    +String city
    +String state
    +int postalCode
    +String country
    -validate()
    +outputAsLabel()  
  }
```

</div>

  [类图]: https://mermaid-js.github.io/mermaid/#/classDiagram

### 使用 ER 图

[ER图] 由实体类型组成，并指定实体之间存在的关系。它描述了特定知识领域中相互关联的事物:

```` markdown title="ER图"
``` mermaid
erDiagram
  CUSTOMER ||--o{ ORDER : places
  ORDER ||--|{ LINE-ITEM : contains
  CUSTOMER }|..|{ DELIVERY-ADDRESS : uses
```
````

<div class="result" markdown>

``` mermaid
erDiagram
  CUSTOMER ||--o{ ORDER : places
  ORDER ||--|{ LINE-ITEM : contains
  CUSTOMER }|..|{ DELIVERY-ADDRESS : uses
```

</div>

  [ER图]: https://mermaid-js.github.io/mermaid/#/entityRelationshipDiagram

### 其他图表类型

除了上面列出的图表类型之外, [Mermaid.js] 对
[pie charts], [gantt charts], [user journeys], [git graphs] and
[requirement diagrams] 提供了支持，这也没有被 Material
for MkDocs 官方支持。这些图应该仍然可以像 [Mermaid.js] 所宣传的那样工作，但我们认为它们不是一个好的选择，主要是因为它们在移动设备上工作不好。

  [pie charts]: https://mermaid-js.github.io/mermaid/#/pie
  [gantt charts]: https://mermaid-js.github.io/mermaid/#/gantt
  [user journeys]: https://mermaid-js.github.io/mermaid/#/user-journey
  [git graphs]: https://mermaid-js.github.io/mermaid/#/gitgraph
  [requirement diagrams]: https://mermaid-js.github.io/mermaid/#/requirementDiagram
