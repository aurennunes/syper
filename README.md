# syper

Um micro-framework web estilo **Express/Fiber**, escrito puramente em **Syntella**, em cima do módulo `http` da stdlib. Roteamento com parâmetros (`/users/:id`), query string, parsing de body JSON, cadeia de middleware e helpers de resposta — em ~uma página de código.

## Instalação

No seu projeto de aplicação (`syt init minha-app`):

```bash
syt add syper path:../syper        # dependência local
# ou, quando publicado:
# syt add github.com/voce/syper
```

Depois `use syper` no código.

## Hello world

```rust
use syper

func main() {
    let app = syper.newApp()

    syper.get(app, "/", func(c) {
        syper.send(c, 200, "ola, mundo")
    })

    syper.listen(app, ":3000")
}
```

```bash
syt run src/main.syt
```

## API

Tudo são **funções livres** que recebem `app` ou `c` (contexto) como primeiro
argumento — veja a nota de design abaixo do porquê.

### Aplicação

| Função | Descrição |
|---|---|
| `syper.newApp()` | cria a aplicação |
| `syper.get(app, pattern, h)` | registra rota GET (idem `post`, `put`, `delete`) |
| `syper.handle(app, method, pattern, h)` | registra rota para um método qualquer |
| `syper.middleware(app, mw)` | adiciona middleware global |
| `syper.listen(app, addr)` | sobe o servidor (ex.: `":3000"`) e bloqueia |

`pattern` aceita parâmetros nomeados: `"/users/:id"`, `"/posts/:slug/comments"`.

### Contexto (`c`) — dentro de um handler

| Função | Retorno |
|---|---|
| `syper.method(c)` | método HTTP (`"GET"`, ...) |
| `syper.path(c)` | caminho da request |
| `syper.param(c, nome)` | parâmetro de rota (`""` se ausente) |
| `syper.queryParam(c, nome)` | parâmetro de query string (`""` se ausente) |
| `syper.body(c)` | corpo cru (string) |
| `syper.bodyJson(c)` | corpo decodificado como JSON (`{}` se vazio) |
| `syper.setHeader(c, nome, valor)` | define header de resposta (antes de send) |
| `syper.send(c, code, texto)` | resposta de texto puro |
| `syper.sendJson(c, code, valor)` | resposta JSON (faz o encode) |

### Middleware

Um middleware é `func(c, next)`. Chame `next()` para continuar a cadeia;
não chamar interrompe (útil p/ auth, etc.).

```rust
// middleware pronto
syper.middleware(app, syper.logger())

// middleware próprio
syper.middleware(app, func(c, next) {
    syper.setHeader(c, "X-Powered-By", "syper")
    next()
})
```

## Exemplo completo

`example/` é um projeto consumidor real (uma API de tarefas CRUD sobre o
syper). Para rodar:

```bash
cd example
syt run src/main.syt          # sobe em :3000 (ou PORT=8080 syt run ...)
```

```bash
curl localhost:3000/todos
curl -X POST -d '{"title":"comprar leite"}' localhost:3000/todos
curl localhost:3000/todos/2
curl -X DELETE localhost:3000/todos/1
```

## Nota de design: por que API funcional e não `app.get(...)`?

No Syntella 0.68.1, blocos `impl ... for` (métodos) só ficam registrados no
**módulo-raiz em execução** — eles **não atravessam a fronteira de um pacote
importado**. Já funções livres, construtores de tipo, leitura de campo e os
métodos da **stdlib** (writer http, map, array) funcionam de qualquer módulo.

Por isso o syper expõe `syper.get(app, ...)` / `syper.sendJson(c, ...)` em vez
de `app.get(...)` / `c.json(...)`. Se/quando o runtime passar a propagar
`impl` de pacotes, dá para adicionar uma camada de métodos por cima sem quebrar
esta API.

## Limitações conhecidas

São **gaps do runtime/stdlib**, não da linguagem:

- **Sem acesso a headers da request** (só dá para escrever headers de resposta) — impede auth via `Authorization`, cookies, content negotiation.
- **Body é sempre string** — sem `multipart/form-data` (upload de arquivo).
- **Servidor roda interpretado** (`syt run`): `http.listenAndServe` ainda não tem backend compilado (`syt build`).
- **Query string não é URL-decoded.**
- Roteamento é por correspondência exata de segmentos (sem wildcard `*` nem rotas opcionais ainda).

## Roadmap curto

- [ ] Grupos de rotas / prefixos (`syper.group(app, "/api")`)
- [ ] Middleware por rota
- [ ] `c.status(code)` encadeável + mais helpers de resposta
- [ ] Router em trie (hoje é varredura linear das rotas)
- [ ] URL-decode de path e query
