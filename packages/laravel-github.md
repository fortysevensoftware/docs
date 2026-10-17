## Contents

- [Introduction](#introduction)
- [When To Use This Package](#when-to-use-this-package)
- [Installation](#installation)
- [GitHub App Setup](#github-app-setup)
  - [Creating and Installing the App](#creating-and-installing-the-app)
  - [Connecting Your Laravel Application](#connecting-your-laravel-application)
  - [Checking Configuration](#checking-configuration)
- [Configuration](#configuration)
- [Repositories](#repositories)
- [Pull Requests](#pull-requests)
  - [Searching Pull Requests](#searching-pull-requests)
  - [Finding a Pull Request by Number](#finding-a-pull-request-by-number)
  - [Retrieving a Pull Request](#retrieving-a-pull-request)
  - [Understanding Pull Request States](#understanding-pull-request-states)
- [Dependency Injection](#dependency-injection)
- [Authentication and Caching](#authentication-and-caching)
- [Error Handling](#error-handling)
- [Testing](#testing)
  - [Running the Tests](#running-the-tests)
  - [Testing Your Own Application](#testing-your-own-application)
- [Local Development](#local-development)
- [Requirements](#requirements)

---

<a name="introduction"></a>
## Introduction

Laravel GitHub is a Laravel package for connecting your application to a GitHub organisation through a GitHub App installation. It provides a straightforward way to browse repositories, search pull requests and retrieve individual pull request details without implementing GitHub App authentication yourself.

The package handles installation token generation, renewal and response caching, allowing your application to focus on how repository and pull request information is presented and used.

```php
use Egough\LaravelGitHub\Facades\GitHub;

// Browse repositories accessible to the installation.
$repositories = GitHub::repositories();

// Search for open pull requests by title.
$results = GitHub::pullRequests('acme/backend', 'checkout', false, 1);

// Retrieve an individual pull request.
$pull = GitHub::pullRequest('acme/backend', 42);
```

---

<a name="when-to-use-this-package"></a>
## When To Use This Package

Use Laravel GitHub when your application needs read-only access to an organisation's repositories and pull requests, particularly when integrating GitHub information into an existing business workflow.

**Common use cases:**

- Linking GitHub repositories to internal projects
- Associating pull requests with support tickets or development tasks
- Displaying pull request titles and current states
- Building repository selectors and pull request search interfaces
- Surfacing development activity in an internal application

**When not to use this package:**

Laravel GitHub currently supports one configured organisation and one GitHub App installation. It does not provide personal access token authentication, user OAuth, multiple connections or webhook handling.

It is also a read-only integration: creating repositories, modifying pull requests and merging code are outside its scope. Your application remains responsible for its own routes, authorization, user interface and storage of any selected repository or pull request links.

---

<a name="installation"></a>
## Installation

Install the package via Composer:

```bash
composer require egough/laravel-github
```

Laravel automatically discovers the package's service provider, so no manual provider registration is required.

### Publishing Configuration

Optionally publish the configuration file:

```bash
php artisan vendor:publish --tag=github-config
```

This creates `config/github.php`, where you can adjust the GitHub connection and caching settings.

No migrations are required.

---

<a name="github-app-setup"></a>
## GitHub App Setup

Laravel GitHub authenticates through a GitHub App installed on the organisation whose repositories you want to access. This requires a GitHub App ID, installation ID and private key.

<a name="creating-and-installing-the-app"></a>
### Creating and Installing the App

1. Create a GitHub App and grant it **Repository metadata: read** and **Pull requests: read** permissions.
2. Install the app on your GitHub organisation, selecting the repositories it should be allowed to access.
3. Generate a private key for the app.
4. Store the private key outside your source repository and publicly accessible web directories, in a location readable by your PHP process.

The package does not require a webhook endpoint or OAuth callback for this integration.

<a name="connecting-your-laravel-application"></a>
### Connecting Your Laravel Application

Add the following settings to your application's `.env` file:

```dotenv
GITHUB_ORGANISATION=acme
GITHUB_APP_ID=123456
GITHUB_INSTALLATION_ID=789012
GITHUB_PRIVATE_KEY_PATH=/absolute/path/to/github-app.pem
```

| Variable | Description |
| --- | --- |
| `GITHUB_ORGANISATION` | The GitHub organisation whose repositories will be browsed. |
| `GITHUB_APP_ID` | The GitHub App ID, not the OAuth client ID. |
| `GITHUB_INSTALLATION_ID` | The ID of the app's installation on the organisation. |
| `GITHUB_PRIVATE_KEY_PATH` | Absolute path to the GitHub App private key file. |

Keep the private key secure and do not commit it to version control.

<a name="checking-configuration"></a>
### Checking Configuration

Use the facade to check whether the required connection settings are present:

```php
use Egough\LaravelGitHub\Facades\GitHub;

$configured = GitHub::configured();
```

`configured()` checks for configuration values only. It does **not** verify that the private key is valid, that GitHub is reachable or that the installation has permission to access a repository.

---

<a name="configuration"></a>
## Configuration

After publishing, the configuration file will be available at `config/github.php`.

```php
return [
    'organisation' => env('GITHUB_ORGANISATION'),
    'app_id' => env('GITHUB_APP_ID'),
    'installation_id' => env('GITHUB_INSTALLATION_ID'),
    'private_key_path' => env('GITHUB_PRIVATE_KEY_PATH'),
    'user_agent' => 'egough-laravel-github',
    'api_version' => '2026-03-10',
    'cache' => [
        'repositories' => 300,
        'pull_requests' => 60,
    ],
];
```

### Options

**`organisation`**

The GitHub organisation associated with the app installation. Repository access is restricted to this configured organisation.

**`app_id`**

The identifier of your GitHub App, used during application authentication.

**`installation_id`**

The installation identifier used to request an installation access token.

**`private_key_path`**

An absolute filesystem path to the app's private key. The PHP process must have permission to read this file.

**`user_agent`**

The User-Agent value sent to GitHub. Defaults to `egough-laravel-github`.

**`api_version`**

The GitHub API version sent with requests. The repository currently specifies `2026-03-10`.

**`cache.repositories`**

How long repository lists are cached, in seconds. Defaults to `300` (five minutes).

**`cache.pull_requests`**

How long pull request searches and individual pull request details are cached, in seconds. Defaults to `60` (one minute).

---

<a name="repositories"></a>
## Repositories

Use the `GitHub` facade to retrieve repositories accessible through the configured GitHub App installation.

```php
use Egough\LaravelGitHub\Facades\GitHub;

$repositories = GitHub::repositories();
```

The method returns repositories belonging to the configured organisation, sorted by full name:

```php
[
    [
        'full_name' => 'acme/backend',
        'url' => 'https://github.com/acme/backend',
    ],
    [
        'full_name' => 'acme/frontend',
        'url' => 'https://github.com/acme/frontend',
    ],
]
```

| Field | Description |
| --- | --- |
| `full_name` | Repository identifier in `owner/repository` format. |
| `url` | The repository's GitHub URL. |

Repository pagination is handled automatically, so you do not need to request each page yourself.

Private repositories are included when the GitHub App installation has access to them. Repositories outside the configured organisation or outside the installation's granted access are not available through the package.

---

<a name="pull-requests"></a>
## Pull Requests

The package provides methods for searching pull requests within an accessible repository and retrieving a specific pull request by number.

<a name="searching-pull-requests"></a>
### Searching Pull Requests

Use `pullRequests()` to search by title:

```php
$results = GitHub::pullRequests(
    repository: 'acme/backend',
    search: 'checkout',
    includeClosed: false,
    page: 1,
);
```

| Argument | Description |
| --- | --- |
| `repository` | Repository full name in `owner/repository` format. |
| `search` | Title search term. An empty search retrieves recently updated pull requests. |
| `includeClosed` | Set to `true` to include closed and merged pull requests. |
| `page` | The search results page to retrieve. |

The result contains the matching pull requests and pagination information:

```php
[
    'data' => [
        [
            'url' => 'https://github.com/acme/backend/pull/42',
            'number' => 42,
            'title' => 'Fix checkout',
            'state' => 'open',
        ],
    ],
    'next_page' => null,
    'incomplete' => false,
]
```

**`data`** contains the matching pull requests. Each item includes the URL, pull request number, title and state.

**`next_page`** identifies the next results page, or is `null` when no further page is available. Pass this value as the `page` argument in a subsequent call to retrieve more results.

**`incomplete`** indicates that GitHub did not return a complete search result. When this is `true`, consider asking users to narrow their search.

Results are ordered by most recently updated, with 30 results per page. GitHub search exposes a maximum of 1,000 results, so applications should validate page input and avoid unbounded searches.

<a name="finding-a-pull-request-by-number"></a>
### Finding a Pull Request by Number

The search method also supports exact-number lookups. Pass either a number such as `42` or a hash-prefixed value such as `#42` as the search term:

```php
$results = GitHub::pullRequests(
    repository: 'acme/backend',
    search: '#42',
    includeClosed: true,
    page: 1,
);
```

The `includeClosed` filter still applies to exact-number searches. If you want to find a merged or closed pull request, set it to `true`.

<a name="retrieving-a-pull-request"></a>
### Retrieving a Pull Request

Use `pullRequest()` when you already know the pull request number:

```php
$pull = GitHub::pullRequest('acme/backend', 42);
```

A matching pull request is returned with its URL, number, title and state:

```php
[
    'url' => 'https://github.com/acme/backend/pull/42',
    'number' => 42,
    'title' => 'Fix checkout',
    'state' => 'open',
]
```

The method returns `null` when the individual pull request cannot be found. Repository access is still restricted to the configured organisation and installation.

<a name="understanding-pull-request-states"></a>
### Understanding Pull Request States

The package returns the following state values:

| State | Description |
| --- | --- |
| `open` | The pull request is currently open. |
| `closed` | The pull request was closed without a confirmed merge. |
| `merged` | The pull request was merged, when GitHub provides merge information in its response. |

This allows you to display a simplified state in your application without interpreting GitHub's raw response yourself.

---

<a name="dependency-injection"></a>
## Dependency Injection

In addition to the facade, you can inject the GitHub client into your controllers or services through Laravel's service container.

```php
use Egough\LaravelGitHub\GitHubClient;

public function __construct(
    private GitHubClient $github,
) {}
```

You can then call the same methods through the injected client:

```php
$repositories = $this->github->repositories();
```

The facade and dependency-injected client resolve the same shared container instance. Choose whichever approach best fits your application's architecture.

---

<a name="authentication-and-caching"></a>
## Authentication and Caching

Laravel GitHub handles GitHub App authentication automatically. It signs a JSON Web Token (JWT) using your app's private key and exchanges that token for an installation access token.

Installation tokens are cached until one minute before their expiry. If GitHub returns a `401` response, the package refreshes the token and retries the request once.

### Response Caching

The package uses Laravel's default cache store:

| Cached Item | Default Duration |
| --- | --- |
| Repository lists | 300 seconds (five minutes) |
| Pull request searches | 60 seconds (one minute) |
| Individual pull request details | 60 seconds (one minute) |
| Installation access tokens | Until one minute before expiry |

Cache keys are scoped to the configured connection settings. Because installation tokens are sensitive credentials, use a protected cache backend with appropriate access controls.

### Request Timeouts

Requests use a three-second connection timeout and a ten-second overall timeout. These limits help prevent an unavailable GitHub API from indefinitely blocking your application.

---

<a name="error-handling"></a>
## Error Handling

The package raises exceptions for connection, authentication and repository access problems, leaving your application responsible for how errors are presented to users.

```php
use Egough\LaravelGitHub\Facades\GitHub;
use Egough\LaravelGitHub\GitHubUnavailable;
use Egough\LaravelGitHub\RepositoryUnavailable;

try {
    $pull = GitHub::pullRequest('acme/backend', 42);
} catch (RepositoryUnavailable $exception) {
    // The repository is outside the configured organisation
    // or the GitHub App installation's access.
} catch (GitHubUnavailable $exception) {
    // Configuration, authentication or an upstream request failed.
}
```

| Exception | Meaning |
| --- | --- |
| `RepositoryUnavailable` | The requested repository is not available to the configured organisation and installation. |
| `GitHubUnavailable` | A configuration, authentication or upstream GitHub request failed. |

A missing **individual pull request** returns `null` rather than throwing `RepositoryUnavailable`.

Upstream response bodies are not included in exception messages.

### Validating Application Input

Your application remains responsible for validating input, authorizing users and throttling any endpoints that expose package functionality.

For browsing endpoints, validate repository identifiers in `owner/repository` format, positive pull request numbers, bounded search strings and search page numbers from 1 to 34.

---

<a name="testing"></a>
## Testing

The package includes tests that fake HTTP requests and use an in-memory cache. GitHub credentials, network access and an external database are not required to run the suite after installing dependencies.

<a name="running-the-tests"></a>
### Running the Tests

Install development dependencies:

```bash
composer install
```

Run the package test suite:

```bash
composer test
```

Or run PHPUnit directly:

```bash
vendor/bin/phpunit
```

<a name="testing-your-own-application"></a>
### Testing Your Own Application

When testing an application that uses Laravel GitHub, you can use Laravel's HTTP testing utilities to fake GitHub API responses:

```php
use Illuminate\Support\Facades\Http;

Http::fake([
    'api.github.com/*' => Http::response([
        'repositories' => [],
    ]),
]);

Http::preventStrayRequests();
```

This is an illustrative HTTP fake; configure the mocked response shape for the specific GitHub endpoint exercised by your test. You may also need to fake authentication endpoints and clear cached values between test cases, depending on the code path being tested.

---

<a name="local-development"></a>
## Local Development

To use a local checkout of Laravel GitHub in another Laravel application, add a Composer path repository to that application's `composer.json`:

```json
{
    "repositories": [
        {
            "type": "path",
            "url": "../packages/laravel-github",
            "options": {
                "symlink": true,
                "versions": {
                    "egough/laravel-github": "^1.0.0"
                }
            }
        }
    ]
}
```

Then require the package:

```bash
composer require egough/laravel-github:^1.0.0
```

Adjust the path to match your local directory layout. With symlinking enabled, PHP source changes in the package checkout become available to the consuming application immediately.

Rerun Composer when package metadata or dependencies change. Before using the application in CI or production, replace the local path dependency with a published release or an accessible Git repository.

---

<a name="requirements"></a>
## Requirements

| Requirement | Version |
| --- | --- |
| PHP | 8.2+ |
| Laravel | 10, 11, 12 or 13 (subject to each version's PHP requirements) |
| PHP extension | OpenSSL |

The package is released under the MIT license.
