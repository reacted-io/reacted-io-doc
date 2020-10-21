# Headline
### Mermaid

```mermaid
graph LR
    A --- B
    B-->C[fa:fa-ban forbidden]
    B-->D(fa:fa-spinner);
```

```mermaid
sequenceDiagram
    participant Razvan
    participant Bob
    Razvan->>Pierre: Hi Pierre!
    Razvan->>John: Hello John!
    loop Action
        John->>John: Thinking
    end
    Note right of John: simple note test!
    John-->>Razvan: Great!
    Pierre->>Razvan: Great!
    John->>Bob: How about you?
    Bob-->>John: Jolly good!
```
