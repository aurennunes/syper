# syper

Um micro-framework web estilo **Express/Fiber**, escrito puramente em **Syntella**, em cima do módulo `http` da stdlib.

- **Router em trie** com parâmetros (`/users/:id`)
- **Grupos de rota** com prefixo e middleware próprios (incl. aninhados)
- **Middleware** global, por grupo e por rota (cadeia `func(c, next)`)
- **URL-decode** de path e query, parsing de body JSON
- Helpers de resposta: `send`, `sendJson`, `status()` encadeável, `redirect`, `html`, `noContent`
- **404/405** automáticos

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
| `syper.get(t, pattern, h)` | registra rota GET (idem `post`, `put`, `delete`) |
| `syper.handle(t, method, pattern, h)` | registra rota para um método qualquer |
| `syper.routeWith(t, method, pattern, mws, h)` | rota com middleware próprio (array `mws`) |
| `syper.group(t, prefix)` | cria um grupo com prefixo (sobre app ou outro grupo) |
| `syper.middleware(t, mw)` | adiciona middleware (funciona em app **e** grupo) |
| `syper.listen(app, addr)` | sobe o servidor (ex.: `":3000"`) e bloqueia |

`t` (target) é um app **ou** um grupo. `pattern` aceita parâmetros nomeados:
`"/users/:id"`, `"/posts/:slug/comments"`.

### Contexto (`c`) — dentro de um handler

| Função | Retorno |
|---|---|
| `syper.method(c)` | método HTTP (`"GET"`, ...) |
| `syper.path(c)` | caminho da request |
| `syper.param(c, nome)` | parâmetro de rota (`""` se ausente) |
| `syper.queryParam(c, nome)` | parâmetro de query string (`""` se ausente) |
| `syper.body(c)` | corpo cru (string) |
| `syper.bodyJson(c)` | corpo decodificado como JSON (`{}` se vazio) |
| `syper.setHeader(c, nome, valor)` | define header de resposta (antes de qualquer write) |
| `syper.status(c, code)` | seta o status corrente e **retorna `c`** (encadeável) |
| `syper.send(c, code, texto)` | resposta de texto com código explícito |
| `syper.sendJson(c, code, valor)` | resposta JSON com código explícito |
| `syper.reply(c, texto)` | resposta de texto usando o status corrente |
| `syper.replyJson(c, valor)` | resposta JSON usando o status corrente |
| `syper.html(c, code, markup)` | resposta `text/html` |
| `syper.redirect(c, code, location)` | redireciona (header `Location`) |
| `syper.noContent(c)` | resposta `204 No Content` |

`status()` é encadeável com `reply`/`replyJson`:

```rust
syper.replyJson(syper.status(c, 201), {"id": 1})
```

### Middleware

Um middleware é `func(c, next)`. Chame `next()` para continuar a cadeia;
não chamar interrompe (útil p/ auth). A ordem de execução é
**global → grupo → rota → handler**.

```rust
// global
syper.middleware(app, syper.logger())

// por grupo
let api = syper.group(app, "/api")
syper.middleware(api, func(c, next) {
    syper.setHeader(c, "X-Powered-By", "syper")
    next()
})
syper.get(api, "/users/:id", func(c) {        // vira /api/users/:id
    syper.sendJson(c, 200, {"id": syper.param(c, "id")})
})

// por rota
syper.routeWith(app, "DELETE", "/posts/:id", [exigirAuth], func(c) {
    syper.noContent(c)
})
```

### Grupos

`syper.group(t, prefix)` devolve um grupo cujas rotas herdam o prefixo e o
middleware do grupo. Grupos podem ser aninhados:

```rust
let api = syper.group(app, "/api")
let v2  = syper.group(api, "/v2")
syper.get(v2, "/ping", func(c) {              // vira /api/v2/ping
    syper.sendJson(c, 200, {"pong": true})
})
```

## Exemplo completo

`example/` é um projeto consumidor real (uma API de tarefas CRUD sobre o
syper). Para rodar:

```bash
cd example
syt run src/main.syt          # sobe em :3000 (ou PORT=8080 syt run ...)
```

Demonstra grupo `/api`, middleware de grupo e de rota, e `status()`:

```bash
curl localhost:3000/api/todos
curl -X POST -d '{"title":"comprar leite"}' localhost:3000/api/todos
curl localhost:3000/api/todos/2
curl -X DELETE "localhost:3000/api/todos/1?confirm=yes"   # middleware de rota exige confirm
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
- **URL-decode trata `%XX` como byte único** — ASCII funciona; sequências multibyte UTF-8 não são recombinadas.
- Roteamento não tem wildcard `*` nem rotas opcionais ainda.

## Roadmap

Itens do roadmap inicial — todos concluídos:

- [x] Grupos de rotas / prefixos (`syper.group(t, "/api")`), incl. aninhados
- [x] Middleware por rota (`syper.routeWith`) e por grupo
- [x] `status(c, code)` encadeável + helpers (`reply`, `replyJson`, `html`, `redirect`, `noContent`)
- [x] Router em **trie** (substituiu a varredura linear)
- [x] URL-decode de path e query

Próximos passos possíveis:

- [ ] Wildcards / rotas catch-all (`/static/*`)
- [ ] Decodificação UTF-8 multibyte no URL-decode
- [ ] Cookies e leitura de headers da request (depende do runtime expor isso)
