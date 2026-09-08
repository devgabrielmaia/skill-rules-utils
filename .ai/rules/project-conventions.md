# Convenções do Projeto

Convenções específicas da stack do Xavier Service (Laravel). Critérios agnósticos de linguagem/framework ficam na rule `code-quality.md`.

## Routes

- `routes/api.php` — rotas de API consumidas pelo frontend do Xavier, autenticação via JWT.
- `routes/integration.php` — rotas de integração usadas por parceiros, autenticação via OAuth2.
- `routes/internal.php` — rotas internas consumidas por outros serviços da Hero Seguros, com checagem de token.

Ao criar uma rota nova, escolher o arquivo pelo consumidor (frontend Xavier, parceiro externo ou serviço interno Hero), não pelo domínio da feature.

## Adapters

- Namespace `App\Adapters`.
- Única camada responsável por comunicação com APIs externas (parceiros, serviços de terceiros).
- Não contêm regra de negócio — apenas request/response e mapeamento de dados externos. Regra de negócio sobre o resultado do Adapter fica no Service que o injeta.

## Enums

- Namespace `App\Enums`.
- Toda entidade que tiver um atributo `*_enum` deve ter seus valores representados em um Enum, para legibilidade do código (nunca usar o valor bruto do banco fora do Enum correspondente).
- O projeto tem dois formatos coexistindo: Enums nativos do PHP (`enum X: string { case ... }`) e classes legadas que estendem `AbstractEnum` com `const`. Em código novo, usar Enum nativo do PHP. Só estender `AbstractEnum` ao adicionar valor a um Enum legado que já usa esse formato — não migrar Enums existentes fora do escopo da tarefa.

## Repositories

- Namespace `App\Repositories`.
- Responsáveis por qualquer acesso a dados que exija integridade — Banco de Dados ou Cache.
- Estendem `HeroLaraToolkit\Abstractions\AbstractRepository` (pacote `hero-seguros/hero-laratoolkit`), que já fornece o CRUD básico. Um Repository novo só precisa declarar `protected string $modelClass` e os métodos de consulta específicos daquele domínio — não reimplementar CRUD já coberto pela abstração.
- Não contêm regra de negócio, apenas acesso e persistência de dados.

## Services

- Namespace `App\Services`, separado por subpasta de domínio (ex.: `App\Services\Agency`, `App\Services\Order`).
- Concentram toda a regra de negócio da aplicação.
- Um Service não deve chamar outro Service — evita dependência cíclica e acoplamento oculto entre regras de negócio. Se duas regras precisam compor, a orquestração é feita por quem já as usa (Controller, Job, Listener), injetando os Services necessários e chamando cada `execute()` na ordem certa.
- Cada Service resolve uma única regra de negócio. O nome da classe reflete esse propósito (ex.: `CreateBilledRequestService`, `GetByIdService`) e expõe um único método público `execute()`.
- Repositories e Adapters podem ser injetados no construtor do Service quando necessário.

## Controllers

- Recebem por injeção de dependência apenas Services — nunca Repository, Adapter ou outra dependência de infraestrutura direto na Controller.
- As dependências são injetadas nos métodos da action (method injection), não no construtor da Controller — assim cada action declara só o que usa, sem inflar o construtor com dependências de outras actions:

```php
public function index(
    \App\Services\Agency\AgencyBilledRequest\ListPendingService $listPendingService
) {
    ...
}
```

- Sempre usar os métodos de resposta da trait `ApiControllerTrait` (herdada pela `Controller` base, do pacote `hero-seguros/hero-laratoolkit`): `returnSuccess()` para sucesso e `returnError()` para erro. Não montar `JsonResponse` manualmente na Controller.
- Validação de entrada via `FormRequest` dedicado (`App\Http\Requests`) e autorização via Policy (`$this->authorize(...)`) — não dentro do Service.

## Tratamento de Erros

- Concentrar o `try/catch` na Controller sempre que possível — é ali que a exceção vira resposta HTTP via `returnError()`/`returnHeroError()`. Um Service não deve capturar uma exceção só para relançá-la sem tratamento algum.
- Quando um Service precisa de um tratamento de erro **localizado** — uma chamada que pode falhar mas tem um caminho alternativo, precisa de log ou retry, sem interromper o restante do fluxo —, preferir o helper nativo `rescue()` a um `try/catch` próprio:

```php
public function execute(array $data): void
{
    $user = rescue(
        fn () => $this->userService->findById($data['userId']),
        function (Throwable $e) {
            Log::warning('Falha ao buscar usuário', ['userId' => $data['userId'], 'exception' => $e]);
            return null;
        }
    );

    if ($user === null) {
        return;
    }

    $payment = rescue(
        fn () => $this->paymentService->charge($user, $data['amount']),
        function (Throwable $e) {
            Log::error('Falha ao cobrar usuário', ['userId' => $user->id, 'exception' => $e]);
            return null;
        }
    );

    if ($payment === null) {
        return;
    }
}
```

- `rescue(callback, rescue = null, report = true)` retorna o resultado do `callback`, ou o valor de `rescue` se uma exceção for lançada — **não** uma tupla `[erro, valor]`. Se `rescue` for uma closure, ela recebe a exceção e decide o que logar e o que devolver; por padrão o helper também reporta a exceção ao Exception Handler (passe `report: false` quando o log já é feito manualmente na closure de fallback, para não duplicar).
- Cuidado com fallback ambíguo: se `null` for um retorno de domínio válido para aquela chamada, não usar `null` como valor de "falhou" — usar um sentinel diferente ou deixar a exceção subir para o `try/catch` da Controller.
- `rescue()` não substitui o `try/catch` da Controller para exceções que devem de fato interromper a request e virar erro HTTP — é para pontos localizados dentro de um Service, onde a falha tem tratamento próprio e o fluxo deve continuar.

## Geral

- Seguir o padrão MVC do Laravel: Controller recebe a requisição e delega; Service concentra a regra de negócio; Model/Repository cuidam dos dados.
- Sempre avaliar se o Laravel já resolve o problema (Eloquent, Form Requests, Policies, Resources, Jobs, Events, Facades, Collections etc.) antes de escrever uma solução própria.
