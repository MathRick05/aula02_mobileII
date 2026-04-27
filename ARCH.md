# ARCH — Arquitetura do Projeto TODO

## Estrutura final de pastas

```
lib/
├── core/
│   └── errors/
│       └── app_error.dart          # Exceção genérica da aplicação
├── features/
│   └── todos/
│       ├── data/
│       │   ├── datasources/
│       │   │   ├── todo_local_datasource.dart   # SharedPreferences (lastSync)
│       │   │   └── todo_remote_datasource.dart  # HTTP → JSONPlaceholder
│       │   ├── models/
│       │   │   └── todo_model.dart              # DTO com fromJson/toJson
│       │   └── repositories/
│       │       └── todo_repository_impl.dart    # Orquestra remote + local
│       ├── domain/
│       │   ├── entities/
│       │   │   └── todo.dart                    # Entidade pura (sem deps)
│       │   └── repositories/
│       │       └── todo_repository.dart         # Contrato abstrato + TodoFetchResult
│       └── presentation/
│           ├── pages/
│           │   └── todos_page.dart              # Tela principal (StatefulWidget)
│           ├── viewmodels/
│           │   └── todo_viewmodel.dart          # ChangeNotifier; depende de TodoRepository
│           └── widgets/
│               └── add_todo_dialog.dart         # Dialog de adição
├── ui/
│   └── app_root.dart               # MaterialApp raiz
└── main.dart                       # Composição root: cria impl e injeta no VM
```

## Fluxo de dependências

```
TodosPage (UI)
    │  context.watch<TodoViewModel>()
    ▼
TodoViewModel (presentation)
    │  depende de TodoRepository (interface do domain)
    │  NÃO conhece Widget nem BuildContext
    ▼
TodoRepository (domain — contrato abstrato)
    │  implementado por
    ▼
TodoRepositoryImpl (data)
    ├──► TodoRemoteDataSource → HTTP → JSONPlaceholder API
    └──► TodoLocalDataSource  → SharedPreferences (persiste lastSync)
```

A injeção de dependência acontece em `main.dart`:
```dart
ChangeNotifierProvider(
  create: (_) => TodoViewModel(TodoRepositoryImpl()),
)
```
O `TodoViewModel` recebe `TodoRepository` (abstrato), portanto nunca conhece a implementação concreta.

## Justificativa da estrutura

**Feature-first** agrupa tudo que pertence à feature `todos` em um único lugar (`features/todos/`), facilitando localizar, modificar e escalar cada feature de forma independente. Dentro de cada feature as camadas seguem Clean Architecture:

| Camada | Responsabilidade |
|---|---|
| `domain` | Regras de negócio puras; sem dependência de Flutter ou bibliotecas externas |
| `data` | Fontes de dados (HTTP, SharedPreferences) e conversão JSON; isola detalhes de infraestrutura |
| `presentation` | UI, ViewModel e widgets; lê estado do ViewModel via Provider |
| `core` | Utilitários e exceções compartilhados entre features |

## Decisões de responsabilidade

**Validação:** feita no `TodoViewModel` — validação de entrada do usuário (título vazio) é uma regra de apresentação, não de domínio.

**Parsing JSON:** encapsulado em `TodoModel.fromJson` na camada `data/models`. A entidade `Todo` (domain) não sabe nada sobre JSON.

**Tratamento de erros:** o `TodoViewModel` captura exceções lançadas pelo repositório e as converte em `errorMessage` (String), expondo apenas estado simples para a UI. O rollback otimista do toggle também é responsabilidade do ViewModel.

**SharedPreferences:** acessado exclusivamente por `TodoLocalDataSource`. A UI nunca chama `SharedPreferences` diretamente — apenas lê `vm.lastSyncLabel`.

**HTTP:** acessado exclusivamente por `TodoRemoteDataSource`. A UI nunca faz chamadas HTTP diretamente.

## Como rodar

```bash
flutter pub get
flutter run
```

