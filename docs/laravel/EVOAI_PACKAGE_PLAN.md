# Evo AI - Laravel Package Implementation Plan

## Overview

This document describes the implementation plan for `evoai/laravel-sdk`, a Laravel package that provides full integration with the Evo AI REST API. The package covers authentication, agent management, chat, sessions, MCP servers, tools, clients, API keys, folders, A2A protocol, and admin operations.

**Package name:** `evoai/laravel-sdk`
**Minimum Laravel version:** 11.x
**Minimum PHP version:** 8.2
**License:** Apache 2.0

### PHP 8.2+ Features Used

- **`readonly class`** — All DTOs are declared as `readonly class`, eliminating the need for `readonly` on each property individually.
- **`true`, `false`, `null` as standalone types** — Used in return types where methods return literal `true` on success or `false` on failure.
- **DNF Types (Disjunctive Normal Form)** — Combined intersection + union types for contracts, e.g., `(Arrayable&Jsonable)|array`.
- **`#[\SensitiveParameter]`** — Applied to method parameters that receive passwords, tokens, or API keys to prevent leaking in stack traces.
- **Constants in traits** — `HasPagination` trait defines `DEFAULT_LIMIT` and `MAX_LIMIT` as trait constants.
- **Enums** — Backed enums for `AgentType`, `SortDirection`, etc.
- **Named arguments** — Used extensively for fluent DTO construction and service method calls.
- **Constructor promotion** — All services and DTOs use constructor promotion for conciseness.
- **Fibers** — Not used directly, but the package is compatible with async Laravel contexts.

### Laravel 11+ Compatibility

- **Auto-discovery** — Service provider auto-discovered via `composer.json` `extra.laravel.providers` (no manual registration in `bootstrap/providers.php` needed).
- **Simplified service provider** — Single `register()` + `boot()`, no kernel or console kernel.
- **Health check integration** — `A2AService::healthCheck()` can be wired into Laravel 11's built-in `/up` health route.
- **`once()` helper** — Used internally for memoizing service instances within the manager class.
- **Pest v3** — Test suite uses Pest v3 (default in Laravel 11) with Arch testing for enforcing code conventions.
- **Config publishing** — Uses `$this->publishes()` in boot; Laravel 11 apps configure via `config/evoai.php` (no `config/*.php` shipped by default).
- **Contextual binding** — Services resolve `EvoAiClient` via Laravel's contextual binding, no static facades needed internally.
- **Dumpable trait** — DTOs use Laravel's `Dumpable` trait for `dd()`/`dump()` support during development.
- **`php artisan make:` not required** — The package provides no Artisan commands; all setup is via config and env variables.

---

## API Analysis Summary

### Authentication
- JWT-based authentication via `Authorization: Bearer {token}`
- Login endpoint returns `access_token` + `token_type`
- Some endpoints also accept `x-api-key` header (chat, shared agents, A2A)

### Multi-tenancy
- Many endpoints require `x-client-id` header (UUID) for client-scoped operations
- Agents, folders, API keys are scoped by client

### Pagination
- Uses `skip`/`limit` pattern (not page/per_page)
- Default: `skip=0`, `limit=100`

### Service Domains (Endpoints)
| Group | Base Path | Auth |
|-------|-----------|------|
| Auth | `/api/v1/auth/` | Public / Bearer |
| Clients | `/api/v1/clients/` | Bearer |
| Agents | `/api/v1/agents/` | Bearer + x-client-id |
| Folders | `/api/v1/agents/folders/` | Bearer + x-client-id |
| API Keys | `/api/v1/agents/apikeys/` | Bearer + x-client-id |
| MCP Servers | `/api/v1/mcp-servers/` | Bearer |
| Tools | `/api/v1/tools/` | Bearer |
| Sessions | `/api/v1/sessions/` | Bearer |
| Chat | `/api/v1/chat/` | Bearer or x-api-key |
| A2A | `/api/v1/a2a/` | x-api-key (optional) |
| Admin | `/api/v1/admin/` | Bearer (admin only) |

---

## Phase 1 - Package Scaffolding & Core Infrastructure

**Goal:** Establish the package structure, configuration, HTTP client, and authentication layer.

### 1.1 - Package Structure

```
evoai/
  laravel-sdk/
    src/
      EvoAiServiceProvider.php
      EvoAiManager.php
      Facades/
        EvoAi.php
      Config/
        evoai.php
      Http/
        EvoAiClient.php
        Middleware/
          LoggingMiddleware.php
      Exceptions/
        EvoAiException.php
        AuthenticationException.php
        ValidationException.php
        NotFoundException.php
        ForbiddenException.php
      Contracts/
        EvoAiClientInterface.php
        Arrayable.php
      DTOs/
        Concerns/
          HasFactory.php
        Auth/
          LoginData.php
          RegisterData.php
          ForgotPasswordData.php
          ResetPasswordData.php
          ChangePasswordData.php
          TokenResponse.php
          UserResponse.php
          MessageResponse.php
        Agent/
          AgentCreateData.php
          AgentUpdateData.php
          AgentData.php
        Client/
          ClientCreateData.php
          ClientUpdateData.php
          ClientData.php
        Session/
          SessionData.php
        Chat/
          ChatRequestData.php
          FileData.php
          ChatResponseData.php
        McpServer/
          McpServerCreateData.php
          McpServerData.php
          ToolConfigData.php
        Tool/
          ToolCreateData.php
          ToolData.php
        Folder/
          FolderCreateData.php
          FolderUpdateData.php
          FolderData.php
        ApiKey/
          ApiKeyCreateData.php
          ApiKeyUpdateData.php
          ApiKeyData.php
        Admin/
          AdminUserCreateData.php
          AuditLogData.php
          AuditLogFilters.php
      Services/
        AuthService.php
        AgentService.php
        ClientService.php
        SessionService.php
        ChatService.php
        McpServerService.php
        ToolService.php
        FolderService.php
        ApiKeyService.php
        AdminService.php
        A2AService.php
      Enums/
        AgentType.php
        SortDirection.php
      Support/
        TokenManager.php
        EvoAiPaginator.php
        EvoAiFake.php
      Traits/
        HasPagination.php
      Events/
        RequestSending.php
        ResponseReceived.php
        RequestFailed.php
        TokenAuthenticated.php
        TokenRefreshed.php
    tests/
      Pest.php
      TestCase.php
      Unit/
        DTOs/
        Support/
      Feature/
        Services/
      Arch/
        ArchTest.php
    composer.json
    phpstan.neon
    pint.json
    README.md
    CHANGELOG.md
```

### 1.2 - `composer.json`

```json
{
    "name": "evoai/laravel-sdk",
    "description": "Laravel SDK for Evo AI API - AI Agent Management",
    "type": "library",
    "license": "Apache-2.0",
    "require": {
        "php": "^8.2",
        "illuminate/contracts": "^11.0",
        "illuminate/http": "^11.0",
        "illuminate/support": "^11.0",
        "illuminate/cache": "^11.0"
    },
    "require-dev": {
        "pestphp/pest": "^3.0",
        "phpstan/phpstan": "^2.0",
        "laravel/pint": "^1.18",
        "orchestra/testbench": "^9.0"
    },
    "autoload": {
        "psr-4": {
            "EvoAi\\LaravelSdk\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "EvoAi\\LaravelSdk\\Tests\\": "tests/"
        }
    },
    "extra": {
        "laravel": {
            "providers": [
                "EvoAi\\LaravelSdk\\EvoAiServiceProvider"
            ],
            "aliases": {
                "EvoAi": "EvoAi\\LaravelSdk\\Facades\\EvoAi"
            }
        }
    },
    "minimum-stability": "stable"
}
```

### 1.3 - Configuration File (`config/evoai.php`)

```php
<?php

return [
    'base_url' => env('EVOAI_BASE_URL', 'http://localhost:8000'),
    'api_prefix' => env('EVOAI_API_PREFIX', '/api/v1'),
    'timeout' => env('EVOAI_TIMEOUT', 30),
    'connect_timeout' => env('EVOAI_CONNECT_TIMEOUT', 10),

    'retry' => [
        'times' => env('EVOAI_RETRY_TIMES', 3),
        'sleep' => env('EVOAI_RETRY_SLEEP', 100), // ms
        'when' => [429, 500, 502, 503, 504], // HTTP status codes to retry
    ],

    // Default credentials (optional, for server-to-server)
    'credentials' => [
        'email' => env('EVOAI_EMAIL'),
        'password' => env('EVOAI_PASSWORD'),
    ],

    // API Key for chat/A2A endpoints
    'api_key' => env('EVOAI_API_KEY'),

    // Default client_id for scoped operations
    'client_id' => env('EVOAI_CLIENT_ID'),

    // Token management
    'token' => [
        'cache_key' => 'evoai_token',
        'cache_store' => env('EVOAI_CACHE_STORE'), // null = default store
        'ttl' => env('EVOAI_TOKEN_TTL', 3300), // seconds (55 min)
        'auto_refresh' => true,
    ],

    // Logging
    'logging' => [
        'enabled' => env('EVOAI_LOGGING', false),
        'channel' => env('EVOAI_LOG_CHANNEL'), // null = default channel
        'redact' => ['password', 'access_token', 'key_value', 'api_key', 'current_password', 'new_password'],
    ],
];
```

### 1.4 - HTTP Client (`EvoAiClient`)

Core HTTP client wrapping Laravel's `Http` facade with:
- Base URL configuration
- Automatic Bearer token injection
- Automatic `x-client-id` header injection when client_id is set
- Automatic `x-api-key` header injection when api_key is set
- Error handling and exception mapping (401 -> AuthenticationException, 404 -> NotFoundException, 422 -> ValidationException, 403 -> ForbiddenException)
- Retry logic with configurable status codes
- `#[\SensitiveParameter]` on credential-handling methods

```php
<?php

namespace EvoAi\LaravelSdk\Http;

use EvoAi\LaravelSdk\Contracts\EvoAiClientInterface;
use EvoAi\LaravelSdk\Exceptions\{
    AuthenticationException,
    ForbiddenException,
    NotFoundException,
    ValidationException,
    EvoAiException,
};
use Illuminate\Http\Client\PendingRequest;
use Illuminate\Support\Facades\Http;

final class EvoAiClient implements EvoAiClientInterface
{
    private ?string $token = null;
    private ?string $apiKey = null;
    private ?string $clientId = null;

    public function __construct(
        private readonly string $baseUrl,
        private readonly string $apiPrefix,
        private readonly int $timeout,
        private readonly int $connectTimeout,
        private readonly array $retryConfig,
    ) {}

    public function authenticate(
        string $email,
        #[\SensitiveParameter] string $password,
    ): self {
        $response = $this->post('/auth/login', [
            'email' => $email,
            'password' => $password,
        ]);

        $this->token = $response['access_token'];

        return $this;
    }

    public function withToken(#[\SensitiveParameter] string $token): self
    {
        $clone = clone $this;
        $clone->token = $token;

        return $clone;
    }

    public function withApiKey(#[\SensitiveParameter] string $apiKey): self
    {
        $clone = clone $this;
        $clone->apiKey = $apiKey;

        return $clone;
    }

    public function withClientId(string $clientId): self
    {
        $clone = clone $this;
        $clone->clientId = $clientId;

        return $clone;
    }

    public function get(string $uri, array $query = []): array { /* ... */ }
    public function post(string $uri, array $data = []): array { /* ... */ }
    public function put(string $uri, array $data = []): array { /* ... */ }
    public function delete(string $uri): void { /* ... */ }
    public function postMultipart(string $uri, array $data, array $files): array { /* ... */ }

    private function buildRequest(): PendingRequest
    {
        $request = Http::baseUrl($this->baseUrl . $this->apiPrefix)
            ->timeout($this->timeout)
            ->connectTimeout($this->connectTimeout)
            ->retry(
                times: $this->retryConfig['times'],
                sleepMilliseconds: $this->retryConfig['sleep'],
                when: fn ($exception, $request) => in_array(
                    $exception->response?->status(),
                    $this->retryConfig['when'],
                    strict: true,
                ),
                throw: false,
            )
            ->acceptJson()
            ->contentType('application/json');

        if ($this->token !== null) {
            $request->withToken($this->token);
        }

        if ($this->apiKey !== null) {
            $request->withHeaders(['x-api-key' => $this->apiKey]);
        }

        if ($this->clientId !== null) {
            $request->withHeaders(['x-client-id' => $this->clientId]);
        }

        return $request;
    }

    private function handleResponse(\Illuminate\Http\Client\Response $response): array
    {
        return match (true) {
            $response->successful() => $response->json() ?? [],
            $response->status() === 401 => throw new AuthenticationException($response),
            $response->status() === 403 => throw new ForbiddenException($response),
            $response->status() === 404 => throw new NotFoundException($response),
            $response->status() === 422 => throw new ValidationException($response),
            default => throw new EvoAiException($response),
        };
    }
}
```

### 1.5 - Service Provider (Laravel 11 style)

```php
<?php

namespace EvoAi\LaravelSdk;

use EvoAi\LaravelSdk\Contracts\EvoAiClientInterface;
use EvoAi\LaravelSdk\Http\EvoAiClient;
use EvoAi\LaravelSdk\Support\TokenManager;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;

final class EvoAiServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->mergeConfigFrom(__DIR__ . '/Config/evoai.php', 'evoai');

        $this->app->singleton(EvoAiClientInterface::class, function (Application $app): EvoAiClient {
            $config = $app['config']['evoai'];

            return new EvoAiClient(
                baseUrl: $config['base_url'],
                apiPrefix: $config['api_prefix'],
                timeout: $config['timeout'],
                connectTimeout: $config['connect_timeout'],
                retryConfig: $config['retry'],
            );
        });

        $this->app->singleton(TokenManager::class, function (Application $app): TokenManager {
            $config = $app['config']['evoai']['token'];

            return new TokenManager(
                cache: $app['cache']->store($config['cache_store']),
                cacheKey: $config['cache_key'],
                ttl: $config['ttl'],
                autoRefresh: $config['auto_refresh'],
            );
        });

        $this->app->singleton(EvoAiManager::class);
        $this->app->alias(EvoAiManager::class, 'evoai');
    }

    public function boot(): void
    {
        if ($this->app->runningInConsole()) {
            $this->publishes([
                __DIR__ . '/Config/evoai.php' => config_path('evoai.php'),
            ], 'evoai-config');
        }
    }
}
```

### 1.6 - Facade

```php
<?php

namespace EvoAi\LaravelSdk\Facades;

use Illuminate\Support\Facades\Facade;

/**
 * @method static \EvoAi\LaravelSdk\Services\AuthService auth()
 * @method static \EvoAi\LaravelSdk\Services\AgentService agents()
 * @method static \EvoAi\LaravelSdk\Services\ClientService clients()
 * @method static \EvoAi\LaravelSdk\Services\FolderService folders()
 * @method static \EvoAi\LaravelSdk\Services\ApiKeyService apiKeys()
 * @method static \EvoAi\LaravelSdk\Services\McpServerService mcpServers()
 * @method static \EvoAi\LaravelSdk\Services\ToolService tools()
 * @method static \EvoAi\LaravelSdk\Services\SessionService sessions()
 * @method static \EvoAi\LaravelSdk\Services\ChatService chat()
 * @method static \EvoAi\LaravelSdk\Services\A2AService a2a()
 * @method static \EvoAi\LaravelSdk\Services\AdminService admin()
 * @method static self authenticate(string $email, string $password)
 * @method static self withToken(string $token)
 * @method static self withApiKey(string $apiKey)
 * @method static self forClient(string $clientId)
 * @method static void fake(array $responses = [])
 *
 * @see \EvoAi\LaravelSdk\EvoAiManager
 */
final class EvoAi extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'evoai';
    }
}
```

### 1.7 - Exception Hierarchy

```php
<?php

namespace EvoAi\LaravelSdk\Exceptions;

use Illuminate\Http\Client\Response;

class EvoAiException extends \RuntimeException
{
    public function __construct(
        public readonly Response $response,
        string $message = '',
    ) {
        parent::__construct(
            $message ?: "Evo AI API error: {$response->status()} - {$response->body()}",
            $response->status(),
        );
    }

    /** @return array<string, mixed> */
    public function context(): array
    {
        return [
            'status' => $this->response->status(),
            'body' => $this->response->json(),
        ];
    }
}

// AuthenticationException (401) - extends EvoAiException
// ForbiddenException     (403) - extends EvoAiException
// NotFoundException      (404) - extends EvoAiException

final class ValidationException extends EvoAiException
{
    /** @return array<array{loc: list<string|int>, msg: string, type: string}> */
    public function errors(): array
    {
        return $this->response->json('detail', []);
    }
}
```

### 1.8 - Deliverables

- [ ] `composer.json` with `^8.2` PHP and `^11.0` illuminate dependencies
- [ ] `EvoAiServiceProvider` with auto-discovery, singleton bindings, config publishing
- [ ] `EvoAi` facade with full PHPDoc `@method` annotations
- [ ] `config/evoai.php` with all sections (retry, token, logging)
- [ ] `EvoAiClient` implementing `EvoAiClientInterface` with `#[\SensitiveParameter]`, immutable `with*()` via clone
- [ ] Exception classes with `context()` method for Laravel's exception reporting
- [ ] `EvoAiClientInterface` contract
- [ ] Pest v3 unit tests for `EvoAiClient` (Http::fake())
- [ ] `phpstan.neon` (level 8), `pint.json` (Laravel preset)

---

## Phase 2 - DTOs (Data Transfer Objects)

**Goal:** Create immutable DTOs using PHP 8.2 `readonly class` with factory methods and Laravel's `Dumpable` trait.

### 2.0 - Base DTO Concern

```php
<?php

namespace EvoAi\LaravelSdk\DTOs\Concerns;

use Illuminate\Contracts\Support\Arrayable;

/** @template TArray of array */
trait HasFactory
{
    /** @param TArray $data */
    abstract public static function from(array $data): static;

    /** @return TArray */
    abstract public function toArray(): array;

    public function toJson(int $options = 0): string
    {
        return json_encode($this->toArray(), $options | JSON_THROW_ON_ERROR);
    }
}
```

All DTOs implement `Arrayable`, `JsonSerializable`, and use `Dumpable` + `HasFactory`.

### 2.1 - Auth DTOs

```php
<?php

namespace EvoAi\LaravelSdk\DTOs\Auth;

use EvoAi\LaravelSdk\DTOs\Concerns\HasFactory;
use Illuminate\Contracts\Support\Arrayable;

readonly class LoginData implements Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(
        public string $email,
        #[\SensitiveParameter]
        public string $password,
    ) {}

    public static function from(array $data): static
    {
        return new static(
            email: $data['email'],
            password: $data['password'],
        );
    }

    public function toArray(): array
    {
        return ['email' => $this->email, 'password' => $this->password];
    }

    public function jsonSerialize(): array { return $this->toArray(); }
}

readonly class RegisterData implements Arrayable, \JsonSerializable { /* ... */ }

readonly class ForgotPasswordData implements Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(public string $email) {}
    // ...
}

readonly class ResetPasswordData implements Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(
        public string $token,
        #[\SensitiveParameter]
        public string $new_password,
    ) {}
    // ...
}

readonly class ChangePasswordData implements Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(
        #[\SensitiveParameter]
        public string $current_password,
        #[\SensitiveParameter]
        public string $new_password,
    ) {}
    // ...
}

readonly class TokenResponse implements Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(
        public string $access_token,
        public string $token_type,
    ) {}
    // ...
}

readonly class UserResponse implements Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(
        public string $id,
        public string $email,
        public bool $is_active,
        public bool $is_admin,
        public ?string $client_id,
        public bool $email_verified,
        public ?string $created_at = null,
        public ?string $updated_at = null,
    ) {}
    // ...
}

readonly class MessageResponse implements Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(public string $message) {}
    // ...
}
```

### 2.2 - Agent DTOs & Enums

```php
<?php

namespace EvoAi\LaravelSdk\Enums;

enum AgentType: string
{
    case LLM = 'llm';
    case Sequential = 'sequential';
    case Parallel = 'parallel';
    case Loop = 'loop';
    case A2A = 'a2a';
    case Workflow = 'workflow';
    case Task = 'task';

    /** @return list<string> */
    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }
}

enum SortDirection: string
{
    case Asc = 'asc';
    case Desc = 'desc';
}
```

```php
<?php

namespace EvoAi\LaravelSdk\DTOs\Agent;

use EvoAi\LaravelSdk\DTOs\Concerns\HasFactory;
use EvoAi\LaravelSdk\Enums\AgentType;

readonly class AgentCreateData implements \Illuminate\Contracts\Support\Arrayable, \JsonSerializable
{
    use HasFactory;

    public function __construct(
        public string $client_id,
        public AgentType $type,
        public ?string $name = null,
        public ?string $description = null,
        public ?string $role = null,
        public ?string $goal = null,
        public ?string $model = null,
        public ?string $api_key_id = null,
        public ?string $instruction = null,
        public ?string $agent_card_url = null,
        public ?string $folder_id = null,
        public ?array $config = null,
    ) {}

    public static function from(array $data): static
    {
        return new static(
            client_id: $data['client_id'],
            type: AgentType::from($data['type']),
            name: $data['name'] ?? null,
            description: $data['description'] ?? null,
            role: $data['role'] ?? null,
            goal: $data['goal'] ?? null,
            model: $data['model'] ?? null,
            api_key_id: $data['api_key_id'] ?? null,
            instruction: $data['instruction'] ?? null,
            agent_card_url: $data['agent_card_url'] ?? null,
            folder_id: $data['folder_id'] ?? null,
            config: $data['config'] ?? null,
        );
    }

    public function toArray(): array
    {
        return array_filter([
            'client_id' => $this->client_id,
            'type' => $this->type->value,
            'name' => $this->name,
            'description' => $this->description,
            'role' => $this->role,
            'goal' => $this->goal,
            'model' => $this->model,
            'api_key_id' => $this->api_key_id,
            'instruction' => $this->instruction,
            'agent_card_url' => $this->agent_card_url,
            'folder_id' => $this->folder_id,
            'config' => $this->config,
        ], fn (mixed $v): bool => $v !== null);
    }

    public function jsonSerialize(): array { return $this->toArray(); }
}

// AgentUpdateData — same pattern but all fields nullable, no client_id/type required

readonly class AgentData implements \Illuminate\Contracts\Support\Arrayable, \JsonSerializable
{
    use HasFactory;
    use \Illuminate\Support\Traits\Dumpable;

    public function __construct(
        public string $id,
        public string $client_id,
        public AgentType $type,
        public ?string $name = null,
        public ?string $description = null,
        public ?string $role = null,
        public ?string $goal = null,
        public ?string $model = null,
        public ?string $api_key_id = null,
        public ?string $instruction = null,
        public ?string $agent_card_url = null,
        public ?string $folder_id = null,
        public ?array $config = null,
        public ?string $created_at = null,
        public ?string $updated_at = null,
    ) {}
    // from(), toArray(), jsonSerialize() ...
}
```

### 2.3 - Client DTOs

```php
readonly class ClientCreateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $name,
        public string $email,
        #[\SensitiveParameter] public string $password,
    ) {}
    // ...
}

readonly class ClientUpdateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(public string $name, public ?string $email = null) {}
    // ...
}

readonly class ClientData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $id,
        public string $name,
        public ?string $email,
        public string $created_at,
    ) {}
    // ...
}
```

### 2.4 - Folder DTOs

```php
readonly class FolderCreateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $name,
        public string $client_id,
        public ?string $description = null,
    ) {}
    // ...
}

readonly class FolderUpdateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(public string $name, public ?string $description = null) {}
    // ...
}

readonly class FolderData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $id,
        public string $client_id,
        public string $name,
        public ?string $description,
        public string $created_at,
        public ?string $updated_at = null,
    ) {}
    // ...
}
```

### 2.5 - API Key DTOs

```php
readonly class ApiKeyCreateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $name,
        public string $provider,
        public string $client_id,
        #[\SensitiveParameter] public string $key_value,
    ) {}
    // ...
}

readonly class ApiKeyUpdateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public ?string $name = null,
        public ?string $provider = null,
        #[\SensitiveParameter] public ?string $key_value = null,
        public ?bool $is_active = null,
    ) {}
    // ...
}

readonly class ApiKeyData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $id,
        public string $client_id,
        public string $name,
        public string $provider,
        public bool $is_active,
        public string $created_at,
        public ?string $updated_at = null,
    ) {}
    // ...
}
```

### 2.6 - MCP Server DTOs

```php
readonly class McpServerCreateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $name,
        public ?string $description = null,
        public string $config_type = 'studio',
        public array $config_json = [],
        public array $environments = [],
        public ?array $tools = null,
        public string $type = 'official',
    ) {}
    // ...
}

readonly class McpServerData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $id,
        public string $name,
        public ?string $description,
        public string $config_type,
        public array $config_json,
        public array $environments,
        public ?array $tools,
        public string $type,
        public string $created_at,
        public ?string $updated_at = null,
    ) {}
    // ...
}

readonly class ToolConfigData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $id,
        public string $name,
        public string $description,
        public array $tags = [],
        public array $examples = [],
        public array $inputModes = [],
        public array $outputModes = [],
    ) {}
    // ...
}
```

### 2.7 - Tool DTOs

```php
readonly class ToolCreateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $name,
        public ?string $description = null,
        public array $config_json = [],
        public array $environments = [],
    ) {}
    // ...
}

readonly class ToolData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $id,
        public string $name,
        public ?string $description,
        public array $config_json,
        public array $environments,
        public string $created_at,
        public ?string $updated_at = null,
    ) {}
    // ...
}
```

### 2.8 - Session DTOs

```php
readonly class SessionData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $id,
        public string $app_name,
        public string $user_id,
        public array $state = [],
        public array $events = [],
        public float $last_update_time = 0.0,
    ) {}
    // ...
}
```

### 2.9 - Chat DTOs

```php
readonly class ChatRequestData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $message,
        public ?string $agent_id = null,
        public ?string $external_id = null,
        /** @var list<FileData>|null */
        public ?array $files = null,
    ) {}
    // ...
}

readonly class FileData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $filename,
        public string $content_type,
        public string $data, // base64 encoded
    ) {}
    // ...
}

readonly class ChatResponseData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $response,
        public array $message_history,
        public string $status,
        public string $timestamp,
    ) {}
    // ...
}
```

### 2.10 - Admin DTOs

```php
readonly class AdminUserCreateData implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public string $email,
        #[\SensitiveParameter] public string $password,
        public string $name,
    ) {}
    // ...
}

readonly class AuditLogData implements Arrayable, \JsonSerializable
{
    use HasFactory, \Illuminate\Support\Traits\Dumpable;
    public function __construct(
        public string $id,
        public string $action,
        public string $resource_type,
        public ?string $resource_id = null,
        public ?array $details = null,
        public ?string $user_id = null,
        public ?string $ip_address = null,
        public ?string $user_agent = null,
        public string $created_at = '',
    ) {}
    // ...
}

readonly class AuditLogFilters implements Arrayable, \JsonSerializable
{
    use HasFactory;
    public function __construct(
        public ?string $user_id = null,
        public ?string $action = null,
        public ?string $resource_type = null,
        public ?string $resource_id = null,
        public ?string $start_date = null,
        public ?string $end_date = null,
        public int $skip = 0,
        public int $limit = 100,
    ) {}

    /** Returns only non-null values as query parameters */
    public function toArray(): array
    {
        return array_filter(get_object_vars($this), fn (mixed $v): bool => $v !== null);
    }

    public function jsonSerialize(): array { return $this->toArray(); }
}
```

### 2.11 - Deliverables

- [ ] All DTOs as `readonly class` with `HasFactory` trait, `Arrayable`, `JsonSerializable`
- [ ] Response DTOs with `Dumpable` trait
- [ ] `#[\SensitiveParameter]` on all password, token, and key fields
- [ ] `AgentType` and `SortDirection` backed enums
- [ ] Pest v3 unit tests for DTO `from()` / `toArray()` round-trip

---

## Phase 3 - Service Classes (Domain Service Layer)

**Goal:** Create service classes that encapsulate all API endpoint interactions, one per domain. Each service receives `EvoAiClient` via constructor injection (resolved by Laravel's container).

### 3.1 - AuthService

```php
<?php

namespace EvoAi\LaravelSdk\Services;

use EvoAi\LaravelSdk\Contracts\EvoAiClientInterface;
use EvoAi\LaravelSdk\DTOs\Auth\{
    ChangePasswordData,
    LoginData,
    MessageResponse,
    RegisterData,
    ResetPasswordData,
    TokenResponse,
    UserResponse,
};

final readonly class AuthService
{
    public function __construct(
        private EvoAiClientInterface $client,
    ) {}

    public function register(RegisterData $data): UserResponse
    {
        return UserResponse::from(
            $this->client->post('/auth/register', $data->toArray()),
        );
    }

    public function registerAdmin(RegisterData $data): UserResponse
    {
        return UserResponse::from(
            $this->client->post('/auth/register-admin', $data->toArray()),
        );
    }

    public function login(LoginData $data): TokenResponse
    {
        return TokenResponse::from(
            $this->client->post('/auth/login', $data->toArray()),
        );
    }

    public function verifyEmail(string $token): MessageResponse
    {
        return MessageResponse::from(
            $this->client->get("/auth/verify-email/{$token}"),
        );
    }

    public function resendVerification(string $email): MessageResponse
    {
        return MessageResponse::from(
            $this->client->post('/auth/resend-verification', ['email' => $email]),
        );
    }

    public function forgotPassword(string $email): MessageResponse
    {
        return MessageResponse::from(
            $this->client->post('/auth/forgot-password', ['email' => $email]),
        );
    }

    public function resetPassword(ResetPasswordData $data): MessageResponse
    {
        return MessageResponse::from(
            $this->client->post('/auth/reset-password', $data->toArray()),
        );
    }

    public function me(): UserResponse
    {
        return UserResponse::from(
            $this->client->post('/auth/me'),
        );
    }

    public function changePassword(ChangePasswordData $data): MessageResponse
    {
        return MessageResponse::from(
            $this->client->post('/auth/change-password', $data->toArray()),
        );
    }
}
```

### 3.2 - ClientService

```php
final readonly class ClientService
{
    use \EvoAi\LaravelSdk\Traits\HasPagination;

    public function __construct(private EvoAiClientInterface $client) {}

    public function create(ClientCreateData $data): ClientData { /* ... */ }

    /** @return list<ClientData> */
    public function list(int $skip = 0, int $limit = 100): array { /* ... */ }

    public function get(string $clientId): ClientData { /* ... */ }
    public function update(string $clientId, ClientUpdateData $data): ClientData { /* ... */ }
    public function delete(string $clientId): void { /* ... */ }
    public function impersonate(string $clientId): TokenResponse { /* ... */ }
}
```

### 3.3 - AgentService

```php
final readonly class AgentService
{
    use \EvoAi\LaravelSdk\Traits\HasPagination;

    public function __construct(private EvoAiClientInterface $client) {}

    public function create(AgentCreateData $data): AgentData { /* ... */ }

    /** @return list<AgentData> */
    public function list(
        string $clientId,
        int $skip = 0,
        int $limit = 100,
        ?string $folderId = null,
        string $sortBy = 'name',
        SortDirection $sortDirection = SortDirection::Asc,
    ): array { /* ... */ }

    public function get(string $agentId, string $clientId): AgentData { /* ... */ }
    public function update(string $agentId, AgentUpdateData|array $data): AgentData { /* ... */ }
    public function delete(string $agentId): void { /* ... */ }

    /** @return array{api_key: string} */
    public function share(string $agentId, string $clientId): array { /* ... */ }
    public function getShared(string $agentId): AgentData { /* ... */ }
    public function assignToFolder(string $agentId, ?string $folderId, string $clientId): AgentData { /* ... */ }

    /** @return list<AgentData> */
    public function import(string $filePath, string $clientId, ?string $folderId = null): array { /* ... */ }
}
```

### 3.4 - FolderService

```php
final readonly class FolderService
{
    use \EvoAi\LaravelSdk\Traits\HasPagination;

    public function __construct(private EvoAiClientInterface $client) {}

    public function create(FolderCreateData $data): FolderData { /* ... */ }
    /** @return list<FolderData> */
    public function list(string $clientId, int $skip = 0, int $limit = 100): array { /* ... */ }
    public function get(string $folderId, string $clientId): FolderData { /* ... */ }
    public function update(string $folderId, FolderUpdateData $data, string $clientId): FolderData { /* ... */ }
    public function delete(string $folderId, string $clientId): void { /* ... */ }
    /** @return list<AgentData> */
    public function listAgents(string $folderId, string $clientId, int $skip = 0, int $limit = 100): array { /* ... */ }
}
```

### 3.5 - ApiKeyService

```php
final readonly class ApiKeyService
{
    use \EvoAi\LaravelSdk\Traits\HasPagination;

    public function __construct(private EvoAiClientInterface $client) {}

    public function create(ApiKeyCreateData $data): ApiKeyData { /* ... */ }
    /** @return list<ApiKeyData> */
    public function list(
        string $clientId,
        int $skip = 0,
        int $limit = 100,
        string $sortBy = 'name',
        SortDirection $sortDirection = SortDirection::Asc,
    ): array { /* ... */ }
    public function get(string $keyId, string $clientId): ApiKeyData { /* ... */ }
    public function update(string $keyId, ApiKeyUpdateData $data, string $clientId): ApiKeyData { /* ... */ }
    public function delete(string $keyId, string $clientId): void { /* ... */ }
}
```

### 3.6 - McpServerService

```php
final readonly class McpServerService
{
    use \EvoAi\LaravelSdk\Traits\HasPagination;

    public function __construct(private EvoAiClientInterface $client) {}

    public function create(McpServerCreateData $data): McpServerData { /* ... */ }
    /** @return list<McpServerData> */
    public function list(int $skip = 0, int $limit = 100): array { /* ... */ }
    public function get(string $serverId): McpServerData { /* ... */ }
    public function update(string $serverId, McpServerCreateData $data): McpServerData { /* ... */ }
    public function delete(string $serverId): void { /* ... */ }
}
```

### 3.7 - ToolService

```php
final readonly class ToolService
{
    use \EvoAi\LaravelSdk\Traits\HasPagination;

    public function __construct(private EvoAiClientInterface $client) {}

    public function create(ToolCreateData $data): ToolData { /* ... */ }
    /** @return list<ToolData> */
    public function list(int $skip = 0, int $limit = 100): array { /* ... */ }
    public function get(string $toolId): ToolData { /* ... */ }
    public function update(string $toolId, ToolCreateData $data): ToolData { /* ... */ }
    public function delete(string $toolId): void { /* ... */ }
}
```

### 3.8 - SessionService

```php
final readonly class SessionService
{
    public function __construct(private EvoAiClientInterface $client) {}

    /** @return list<SessionData> */
    public function getClientSessions(string $clientId): array { /* ... */ }
    /** @return list<SessionData> */
    public function getAgentSessions(string $agentId, int $skip = 0, int $limit = 100): array { /* ... */ }
    public function get(string $sessionId): SessionData { /* ... */ }
    public function delete(string $sessionId): void { /* ... */ }
    public function getMessages(string $sessionId): array { /* ... */ }
}
```

### 3.9 - ChatService

```php
final readonly class ChatService
{
    public function __construct(private EvoAiClientInterface $client) {}

    public function send(string $agentId, string $externalId, ChatRequestData $data): ChatResponseData
    {
        return ChatResponseData::from(
            $this->client->post("/chat/{$agentId}/{$externalId}", $data->toArray()),
        );
    }
}
```

### 3.10 - A2AService

```php
final readonly class A2AService
{
    public function __construct(private EvoAiClientInterface $client) {}

    public function sendMessage(string $agentId, array $jsonRpcPayload, #[\SensitiveParameter] ?string $apiKey = null): array { /* ... */ }
    public function getAgentCard(string $agentId): array { /* ... */ }
    public function healthCheck(): array { /* ... */ }
    public function listSessions(string $agentId, string $externalId, #[\SensitiveParameter] ?string $apiKey = null): array { /* ... */ }
    public function getSessionHistory(string $agentId, string $sessionId, int $limit = 50, #[\SensitiveParameter] ?string $apiKey = null): array { /* ... */ }
    public function getConversationHistory(string $agentId, #[\SensitiveParameter] ?string $apiKey = null): array { /* ... */ }
}
```

### 3.11 - AdminService

```php
final readonly class AdminService
{
    public function __construct(private EvoAiClientInterface $client) {}

    /** @return list<AuditLogData> */
    public function getAuditLogs(AuditLogFilters $filters): array { /* ... */ }
    /** @return list<UserResponse> */
    public function listUsers(int $skip = 0, int $limit = 100): array { /* ... */ }
    public function createUser(AdminUserCreateData $data): UserResponse { /* ... */ }
    public function deactivateUser(string $userId): void { /* ... */ }
}
```

### 3.12 - Main EvoAiManager Class

```php
<?php

namespace EvoAi\LaravelSdk;

use EvoAi\LaravelSdk\Contracts\EvoAiClientInterface;
use EvoAi\LaravelSdk\Services\{
    AuthService, AgentService, ClientService, FolderService,
    ApiKeyService, McpServerService, ToolService, SessionService,
    ChatService, A2AService, AdminService,
};

final class EvoAiManager
{
    private EvoAiClientInterface $client;

    public function __construct(
        private readonly EvoAiClientInterface $baseClient,
    ) {
        $this->client = $this->baseClient;
    }

    public function auth(): AuthService { return once(fn () => new AuthService($this->client)); }
    public function clients(): ClientService { return once(fn () => new ClientService($this->client)); }
    public function agents(): AgentService { return once(fn () => new AgentService($this->client)); }
    public function folders(): FolderService { return once(fn () => new FolderService($this->client)); }
    public function apiKeys(): ApiKeyService { return once(fn () => new ApiKeyService($this->client)); }
    public function mcpServers(): McpServerService { return once(fn () => new McpServerService($this->client)); }
    public function tools(): ToolService { return once(fn () => new ToolService($this->client)); }
    public function sessions(): SessionService { return once(fn () => new SessionService($this->client)); }
    public function chat(): ChatService { return once(fn () => new ChatService($this->client)); }
    public function a2a(): A2AService { return once(fn () => new A2AService($this->client)); }
    public function admin(): AdminService { return once(fn () => new AdminService($this->client)); }

    public function authenticate(
        string $email,
        #[\SensitiveParameter] string $password,
    ): self {
        $this->client->authenticate($email, $password);
        return $this;
    }

    public function withToken(#[\SensitiveParameter] string $token): self
    {
        $new = clone $this;
        $new->client = $this->client->withToken($token);
        return $new;
    }

    public function withApiKey(#[\SensitiveParameter] string $apiKey): self
    {
        $new = clone $this;
        $new->client = $this->client->withApiKey($apiKey);
        return $new;
    }

    public function forClient(string $clientId): self
    {
        $new = clone $this;
        $new->client = $this->client->withClientId($clientId);
        return $new;
    }
}
```

### 3.13 - Deliverables

- [ ] 11 Service classes as `final readonly class` with constructor injection
- [ ] `EvoAiManager` class using `once()` for service memoization
- [ ] All services use typed DTOs for inputs/outputs
- [ ] `SortDirection` enum used instead of raw strings
- [ ] `#[\SensitiveParameter]` on credential parameters across all services
- [ ] Service Provider binds `EvoAiManager` as singleton
- [ ] Pest v3 feature tests for each service using `Http::fake()`

---

## Phase 4 - Pagination Support

**Goal:** Implement a pagination helper that adapts the `skip`/`limit` API pattern to Laravel's conventions.

### 4.1 - HasPagination Trait

```php
<?php

namespace EvoAi\LaravelSdk\Traits;

use EvoAi\LaravelSdk\Support\EvoAiPaginator;

trait HasPagination
{
    const int DEFAULT_LIMIT = 100;
    const int MAX_LIMIT = 1000;
}
```

### 4.2 - Paginator Class

```php
<?php

namespace EvoAi\LaravelSdk\Support;

/**
 * @template TItem
 * @implements \IteratorAggregate<int, TItem>
 */
final readonly class EvoAiPaginator implements \IteratorAggregate, \Countable, \JsonSerializable
{
    /**
     * @param list<TItem> $items
     */
    public function __construct(
        private array $items,
        private int $skip,
        private int $limit,
        private ?int $total = null,
    ) {}

    /** @return list<TItem> */
    public function items(): array { return $this->items; }
    public function count(): int { return count($this->items); }
    public function isEmpty(): bool { return $this->items === []; }
    public function isNotEmpty(): bool { return !$this->isEmpty(); }
    public function hasMore(): bool { return $this->count() >= $this->limit; }
    public function nextSkip(): int { return $this->skip + $this->limit; }
    public function perPage(): int { return $this->limit; }
    public function currentPage(): int { return (int) floor($this->skip / max($this->limit, 1)) + 1; }
    public function total(): ?int { return $this->total; }

    /** @return \ArrayIterator<int, TItem> */
    public function getIterator(): \ArrayIterator { return new \ArrayIterator($this->items); }

    public function jsonSerialize(): array
    {
        return [
            'data' => $this->items,
            'meta' => [
                'skip' => $this->skip,
                'limit' => $this->limit,
                'current_page' => $this->currentPage(),
                'has_more' => $this->hasMore(),
                'total' => $this->total,
            ],
        ];
    }
}
```

### 4.3 - Fluent Pagination on Services

```php
// Usage:
$agents = EvoAi::agents()->list($clientId, skip: 0, limit: 20);

// With paginator helper:
$paginator = EvoAi::agents()->paginate($clientId, page: 1, perPage: 20);
$paginator->items();       // AgentData[]
$paginator->hasMore();     // bool
$paginator->currentPage(); // 1
$paginator->nextSkip();    // 20

// Iterate directly:
foreach ($paginator as $agent) {
    echo $agent->name;
}
```

### 4.4 - Deliverables

- [ ] `EvoAiPaginator<TItem>` as generic `readonly class`
- [ ] `HasPagination` trait with `const int` (PHP 8.2 trait constants)
- [ ] `paginate()` method on applicable services
- [ ] Pest v3 unit tests for pagination logic

---

## Phase 5 - Token Management & Caching

**Goal:** Implement automatic token management with caching to avoid re-authenticating on every request.

### 5.1 - Token Manager

```php
<?php

namespace EvoAi\LaravelSdk\Support;

use Illuminate\Contracts\Cache\Repository as CacheRepository;

final class TokenManager
{
    public function __construct(
        private readonly CacheRepository $cache,
        private readonly string $cacheKey,
        private readonly int $ttl,
        private readonly bool $autoRefresh,
    ) {}

    public function getToken(): ?string
    {
        return $this->cache->get($this->cacheKey);
    }

    public function setToken(
        #[\SensitiveParameter] string $token,
        ?int $ttl = null,
    ): void {
        $this->cache->put($this->cacheKey, $token, $ttl ?? $this->ttl);
    }

    public function clearToken(): void
    {
        $this->cache->forget($this->cacheKey);
    }

    public function hasToken(): bool
    {
        return $this->cache->has($this->cacheKey);
    }

    public function shouldAutoRefresh(): bool
    {
        return $this->autoRefresh;
    }
}
```

### 5.2 - Behavior

- On first request requiring auth, authenticate using configured credentials
- Cache the JWT token using Laravel's cache (configurable store)
- Configurable TTL (default: 55 minutes, assuming 1-hour JWT expiry)
- On 401 response, clear cache and re-authenticate once (prevent infinite loops)
- Support for multiple named connections (e.g., different credentials per client)

### 5.3 - Deliverables

- [ ] `TokenManager` class with `#[\SensitiveParameter]` on `setToken()`
- [ ] Auto-authentication on `EvoAiClient` first request
- [ ] Auto-refresh on 401
- [ ] Config options for cache store and TTL
- [ ] Pest v3 unit tests for token lifecycle

---

## Phase 6 - Events & Logging

**Goal:** Dispatch Laravel events on API interactions and provide structured logging.

### 6.1 - Events

```php
<?php

namespace EvoAi\LaravelSdk\Events;

readonly class RequestSending
{
    public function __construct(
        public string $method,
        public string $url,
        public ?array $payload = null,
    ) {}
}

readonly class ResponseReceived
{
    public function __construct(
        public string $method,
        public string $url,
        public int $statusCode,
        public ?array $response,
        public float $durationMs,
    ) {}
}

readonly class RequestFailed
{
    public function __construct(
        public string $method,
        public string $url,
        public int $statusCode,
        public ?array $response,
        public float $durationMs,
        public \Throwable $exception,
    ) {}
}

readonly class TokenAuthenticated
{
    public function __construct(
        public string $email,
        public float $durationMs,
    ) {}
}

readonly class TokenRefreshed
{
    public function __construct(
        public float $durationMs,
    ) {}
}
```

### 6.2 - Logging Middleware

```php
<?php

namespace EvoAi\LaravelSdk\Http\Middleware;

use Illuminate\Support\Facades\Log;

final readonly class LoggingMiddleware
{
    public function __construct(
        private bool $enabled,
        private ?string $channel,
        private array $redactKeys,
    ) {}

    public function __invoke(callable $handler): callable
    {
        return function (string $method, string $url, array $options) use ($handler) {
            if (! $this->enabled) {
                return $handler($method, $url, $options);
            }

            $start = hrtime(true);
            // ... log, redact sensitive fields, measure duration
        };
    }

    private function redact(array $data): array
    {
        return collect($data)->map(function (mixed $value, string $key): mixed {
            if (in_array($key, $this->redactKeys, strict: true)) {
                return '********';
            }
            return is_array($value) ? $this->redact($value) : $value;
        })->all();
    }
}
```

### 6.3 - Deliverables

- [ ] 5 Event classes as `readonly class`
- [ ] `LoggingMiddleware` with sensitive data redaction
- [ ] `hrtime(true)` for nanosecond-precision duration measurement
- [ ] Pest v3 unit tests for event dispatching and redaction

---

## Phase 7 - Testing Infrastructure

**Goal:** Provide testing utilities so package consumers can easily mock the Evo AI API in their tests.

### 7.1 - Fake Client

```php
<?php

namespace EvoAi\LaravelSdk\Support;

use EvoAi\LaravelSdk\EvoAiManager;

final class EvoAiFake extends EvoAiManager
{
    /** @var array<string, list<array{data: mixed, callable: ?callable}>> */
    private array $stubbedResponses = [];

    /** @var array<string, list<array{args: array}>> */
    private array $recorded = [];

    public static function create(array $responses = []): self { /* ... */ }

    public function assertSent(string $service, ?callable $callback = null): void { /* ... */ }
    public function assertSentCount(string $service, int $count): void { /* ... */ }
    public function assertNotSent(string $service): void { /* ... */ }
    public function assertNothingSent(): void { /* ... */ }
}
```

### 7.2 - Usage

```php
// In tests (Pest v3):
use EvoAi\LaravelSdk\Facades\EvoAi;
use EvoAi\LaravelSdk\DTOs\Agent\AgentData;
use EvoAi\LaravelSdk\DTOs\Auth\TokenResponse;

it('lists agents', function () {
    EvoAi::fake([
        'agents.list' => [
            AgentData::from([...]),
            AgentData::from([...]),
        ],
        'auth.login' => TokenResponse::from([
            'access_token' => 'fake-token',
            'token_type' => 'bearer',
        ]),
    ]);

    $agents = EvoAi::agents()->list('client-uuid');

    expect($agents)->toHaveCount(2);
    EvoAi::assertSent('agents.list');
    EvoAi::assertNotSent('agents.delete');
});

// Sequence responses:
EvoAi::fake([
    'auth.login' => EvoAi::sequence([
        TokenResponse::from(['access_token' => 'token-1', 'token_type' => 'bearer']),
        new AuthenticationException('Invalid credentials'),
    ]),
]);
```

### 7.3 - Arch Tests (Pest v3)

```php
// tests/Arch/ArchTest.php
arch('DTOs are readonly classes')
    ->expect('EvoAi\LaravelSdk\DTOs')
    ->toBeReadonly();

arch('Services are final readonly classes')
    ->expect('EvoAi\LaravelSdk\Services')
    ->toBeFinal()
    ->toBeReadonly();

arch('Enums are backed string enums')
    ->expect('EvoAi\LaravelSdk\Enums')
    ->toBeEnums();

arch('No debugging functions')
    ->expect(['dd', 'dump', 'ray'])
    ->not->toBeUsed();

arch('DTOs implement Arrayable')
    ->expect('EvoAi\LaravelSdk\DTOs')
    ->toImplement(\Illuminate\Contracts\Support\Arrayable::class);
```

### 7.4 - Deliverables

- [ ] `EvoAiFake` class implementing the same interface as `EvoAiManager`
- [ ] `EvoAi::fake()` static method on facade
- [ ] Assertion methods (`assertSent`, `assertSentCount`, `assertNotSent`, `assertNothingSent`)
- [ ] Sequence support for multi-call scenarios
- [ ] Pest v3 Arch tests for code convention enforcement

---

## Phase 8 - Documentation & Publishing

**Goal:** Final documentation, examples, and package publishing.

### 8.1 - README.md

- Installation (`composer require evoai/laravel-sdk`)
- Configuration (auto-discovery, `php artisan vendor:publish --tag=evoai-config`)
- Quick start example
- Authentication patterns (credentials, token, API key)
- Service usage examples (CRUD for each service)
- Pagination
- Error handling
- Testing utilities (`EvoAi::fake()`)
- Contributing guide

### 8.2 - Usage Examples

```php
// Basic authentication
$evoai = app(\EvoAi\LaravelSdk\EvoAiManager::class);
$evoai->authenticate('user@example.com', 'password');

// Or use facade
use EvoAi\LaravelSdk\Facades\EvoAi;

EvoAi::authenticate('user@example.com', 'password');

// List agents for a client
$agents = EvoAi::forClient($clientId)->agents()->list(
    clientId: $clientId,
    sortDirection: SortDirection::Desc,
);

// Create an agent
use EvoAi\LaravelSdk\DTOs\Agent\AgentCreateData;
use EvoAi\LaravelSdk\Enums\AgentType;

$agent = EvoAi::agents()->create(new AgentCreateData(
    client_id: $clientId,
    type: AgentType::LLM,
    name: 'my_agent',
    description: 'A helpful assistant',
    model: 'gemini-2.0-flash',
    api_key_id: $apiKeyId,
    instruction: 'You are a helpful assistant.',
));

// Chat with an agent
use EvoAi\LaravelSdk\DTOs\Chat\ChatRequestData;

$response = EvoAi::chat()->send(
    agentId: $agentId,
    externalId: $externalId,
    data: new ChatRequestData(message: 'Hello!'),
);
echo $response->response;

// Paginate
$paginator = EvoAi::agents()->paginate($clientId, page: 2, perPage: 20);
foreach ($paginator as $agent) {
    echo "{$agent->name}: {$agent->type->value}\n";
}

// Manage MCP Servers
$servers = EvoAi::mcpServers()->list();

// A2A Protocol
$card = EvoAi::a2a()->getAgentCard($agentId);

// Admin operations
use EvoAi\LaravelSdk\DTOs\Admin\AuditLogFilters;

$logs = EvoAi::admin()->getAuditLogs(new AuditLogFilters(
    action: 'create',
    limit: 50,
));
```

### 8.3 - Deliverables

- [ ] Complete README.md
- [ ] CHANGELOG.md
- [ ] CONTRIBUTING.md
- [ ] Packagist submission preparation
- [ ] GitHub Actions CI workflow:
  - `laravel/pint` for code style
  - `phpstan/phpstan` level 8 for static analysis
  - `pestphp/pest` v3 for tests
  - Matrix: PHP 8.2, 8.3, 8.4 x Laravel 11, 12

---

## Implementation Order & Dependencies

```
Phase 1 (Core)
  └──> Phase 2 (DTOs)
         └──> Phase 3 (Services)
                ├──> Phase 4 (Pagination)
                ├──> Phase 5 (Token Management)
                └──> Phase 6 (Events & Logging)
                       └──> Phase 7 (Testing)
                              └──> Phase 8 (Documentation)
```

### Estimated File Count by Phase

| Phase | Files | Description |
|-------|-------|-------------|
| 1 | ~14 | Core infra, client, contract, exceptions, config, provider |
| 2 | ~35 | DTOs for all 11 domains + enums + concern |
| 3 | ~13 | 11 services + manager + trait |
| 4 | ~3 | Paginator, trait update |
| 5 | ~2 | TokenManager, config update |
| 6 | ~7 | 5 events, logging middleware, config update |
| 7 | ~4 | Fake, assertions, sequences, arch tests |
| 8 | ~5 | README, CHANGELOG, CONTRIBUTING, CI, pint.json |
| **Total** | **~83** | |

---

## Technical Decisions & Rationale

### Why `readonly class` (PHP 8.2) instead of `readonly` per property?
- Entire class is immutable by declaration — no accidental mutable properties
- Cleaner syntax, fewer annotations, enforced at the class level
- Prevents dynamic properties (deprecated in PHP 8.2)

### Why `#[\SensitiveParameter]`?
- Prevents passwords, tokens, and API keys from appearing in stack traces, logs, and error reports
- Native PHP 8.2 attribute — no library needed
- Applied to: `authenticate()`, `withToken()`, `withApiKey()`, all password/key DTO constructors

### Why `final readonly class` for services?
- `final` prevents inheritance — services should be composed, not extended
- `readonly` guarantees immutability — the injected client cannot be swapped after construction
- Combined with constructor promotion, each service is a minimal, focused class

### Why `once()` in Manager?
- Laravel 11's `once()` helper memoizes the result within the same request lifecycle
- Avoids creating multiple service instances when calling `EvoAi::agents()` repeatedly
- Automatically reset between requests in long-running processes (Octane-safe)

### Why Pest v3 and Arch tests?
- Pest v3 is the default test framework in Laravel 11
- Arch tests enforce structural conventions (readonly DTOs, final services, backed enums) as automated tests
- Prevents regressions in code style as the package evolves

### Why not use Eloquent models?
- The package is a pure API client, not tied to a database
- DTOs are simpler and more appropriate for API responses

### Why skip/limit instead of page/perPage internally?
- The Evo AI API uses skip/limit natively
- The `paginate()` convenience method translates page/perPage to skip/limit for Laravel developers

### Why Laravel Http facade?
- Already available in all Laravel apps
- Built-in retry, fake, recording, and macro support
- Follows Laravel conventions for HTTP clients

### Why events instead of callbacks?
- Decoupled architecture
- Multiple listeners can react to the same event
- Follows Laravel's event-driven conventions
- Easy to extend (e.g., metrics, monitoring) without modifying package code

---

## PHP 8.2+ Features Summary

| Feature | Where Used |
|---------|------------|
| `readonly class` | All DTOs, Paginator, Event classes |
| `final readonly class` | All Service classes |
| `#[\SensitiveParameter]` | Passwords, tokens, API keys in Client, DTOs, Services |
| `const` in traits | `HasPagination::DEFAULT_LIMIT`, `HasPagination::MAX_LIMIT` |
| Backed enums | `AgentType`, `SortDirection` |
| `true`/`false`/`null` standalone types | Exception helpers, validation methods |
| DNF types | `AgentUpdateData|array` in service methods |
| Named arguments | DTO constructors, service methods, config |
| Constructor promotion | All classes |
| `match` expressions | `EvoAiClient::handleResponse()` |
| First-class callable syntax | Retry `when` callback, `array_filter` callbacks |

## Laravel 11+ Features Summary

| Feature | Where Used |
|---------|------------|
| Auto-discovery | `composer.json` extra.laravel.providers |
| `once()` helper | `EvoAiManager` service memoization |
| `Dumpable` trait | Response DTOs |
| Pest v3 | All tests + Arch tests |
| Contextual binding | Service Provider binds `EvoAiClientInterface` |
| Config store (not driver) | Token cache uses `cache_store` key |
| Simplified ServiceProvider | No kernels, no console kernel |

---

## API Endpoint Coverage Matrix

| Endpoint | Method | Service | Method Name |
|----------|--------|---------|-------------|
| `/auth/register` | POST | AuthService | `register()` |
| `/auth/register-admin` | POST | AuthService | `registerAdmin()` |
| `/auth/login` | POST | AuthService | `login()` |
| `/auth/verify-email/{token}` | GET | AuthService | `verifyEmail()` |
| `/auth/resend-verification` | POST | AuthService | `resendVerification()` |
| `/auth/forgot-password` | POST | AuthService | `forgotPassword()` |
| `/auth/reset-password` | POST | AuthService | `resetPassword()` |
| `/auth/me` | POST | AuthService | `me()` |
| `/auth/change-password` | POST | AuthService | `changePassword()` |
| `/clients` | POST | ClientService | `create()` |
| `/clients/` | GET | ClientService | `list()` |
| `/clients/{id}` | GET | ClientService | `get()` |
| `/clients/{id}` | PUT | ClientService | `update()` |
| `/clients/{id}` | DELETE | ClientService | `delete()` |
| `/clients/{id}/impersonate` | POST | ClientService | `impersonate()` |
| `/agents/` | GET | AgentService | `list()` |
| `/agents/` | POST | AgentService | `create()` |
| `/agents/{id}` | GET | AgentService | `get()` |
| `/agents/{id}` | PUT | AgentService | `update()` |
| `/agents/{id}` | DELETE | AgentService | `delete()` |
| `/agents/{id}/share` | POST | AgentService | `share()` |
| `/agents/{id}/shared` | GET | AgentService | `getShared()` |
| `/agents/{id}/folder` | PUT | AgentService | `assignToFolder()` |
| `/agents/import` | POST | AgentService | `import()` |
| `/agents/folders` | POST | FolderService | `create()` |
| `/agents/folders` | GET | FolderService | `list()` |
| `/agents/folders/{id}` | GET | FolderService | `get()` |
| `/agents/folders/{id}` | PUT | FolderService | `update()` |
| `/agents/folders/{id}` | DELETE | FolderService | `delete()` |
| `/agents/folders/{id}/agents` | GET | FolderService | `listAgents()` |
| `/agents/apikeys` | POST | ApiKeyService | `create()` |
| `/agents/apikeys` | GET | ApiKeyService | `list()` |
| `/agents/apikeys/{id}` | GET | ApiKeyService | `get()` |
| `/agents/apikeys/{id}` | PUT | ApiKeyService | `update()` |
| `/agents/apikeys/{id}` | DELETE | ApiKeyService | `delete()` |
| `/mcp-servers/` | POST | McpServerService | `create()` |
| `/mcp-servers/` | GET | McpServerService | `list()` |
| `/mcp-servers/{id}` | GET | McpServerService | `get()` |
| `/mcp-servers/{id}` | PUT | McpServerService | `update()` |
| `/mcp-servers/{id}` | DELETE | McpServerService | `delete()` |
| `/tools/` | POST | ToolService | `create()` |
| `/tools/` | GET | ToolService | `list()` |
| `/tools/{id}` | GET | ToolService | `get()` |
| `/tools/{id}` | PUT | ToolService | `update()` |
| `/tools/{id}` | DELETE | ToolService | `delete()` |
| `/sessions/client/{id}` | GET | SessionService | `getClientSessions()` |
| `/sessions/agent/{id}` | GET | SessionService | `getAgentSessions()` |
| `/sessions/{id}` | GET | SessionService | `get()` |
| `/sessions/{id}` | DELETE | SessionService | `delete()` |
| `/sessions/{id}/messages` | GET | SessionService | `getMessages()` |
| `/chat/{agentId}/{externalId}` | POST | ChatService | `send()` |
| `/a2a/{agentId}` | POST | A2AService | `sendMessage()` |
| `/a2a/{agentId}/.well-known/agent.json` | GET | A2AService | `getAgentCard()` |
| `/a2a/health` | GET | A2AService | `healthCheck()` |
| `/a2a/{agentId}/sessions` | GET | A2AService | `listSessions()` |
| `/a2a/{agentId}/sessions/{id}/history` | GET | A2AService | `getSessionHistory()` |
| `/a2a/{agentId}/conversation/history` | POST | A2AService | `getConversationHistory()` |
| `/admin/audit-logs` | GET | AdminService | `getAuditLogs()` |
| `/admin/users` | GET | AdminService | `listUsers()` |
| `/admin/users` | POST | AdminService | `createUser()` |
| `/admin/users/{id}` | DELETE | AdminService | `deactivateUser()` |

**Total: 56 endpoints covered across 11 service classes.**
