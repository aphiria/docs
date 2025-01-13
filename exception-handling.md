<h1 id="doc-title">Exception Handling</h1>

<nav class="toc-nav">

<div class="toc-nav-contents">

<h2 id="table-of-contents">Table of Contents</h2>

<ol>
<li><a href="#global-exception-handler">Global Exception Handler</a></li>
<li><a href="#problem-details-exception-renderer">Problem Details Exception Renderer</a><ol>
<li><a href="#custom-problem-details-mappings">Custom Problem Details Mappings</a></li>
</ol>
</li>
<li><a href="#console-exception-renderer">Console Exception Renderer</a><ol>
<li><a href="#output-writers">Output Writers</a></li>
</ol>
</li>
<li><a href="#logging">Logging</a><ol>
<li><a href="#exception-log-levels">Exception Log Levels</a></li>
</ol>
</li>
</ol>

</div>

</nav>

<h2 id="global-exception-handler">Global Exception Handler</h2>

At some point, your application is going to throw an unhandled exception or shut down unexpectedly.  When this happens, it would be nice to log details about the error and present a nicely-formatted response for the user.  Aphiria provides `GlobalExceptionHandler` to do just this.  It can be used to render exceptions for both HTTP and console applications, and is framework-agnostic.

<div class="context-framework" markdown="1">

To learn more about how to configure exceptions in modules, read the [configuration documentation](configuration.md#component-exception-handler).

</div>
<div class="context-library" markdown="1">

Let's review how to manually configure the exception handler.

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ProblemDetailsExceptionRenderer;

$exceptionRenderer = new ProblemDetailsExceptionRenderer();
$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
// This registers the handler as the default handler in PHP
$globalExceptionHandler->registerWithPhp();
```

That's it.  Now, whenever an unhandled error or exception is thrown, the global exception handler will catch it, [log it](#logging), and [render it](#problem-details-exception-renderer).  We'll go into more details on how to customize it below.

</div>

<h2 id="problem-details-exception-renderer">Problem Details Exception Renderer</h2>

<div class="context-framework" markdown="1">

> **Note:** Please refer to the [configuration documentation](configuration.md#component-exception-handler) to learn more about configuring problem details for exceptions.

</div>

`ProblemDetailsExceptionRenderer` is provided out of the box to simplify rendering <a href="https://tools.ietf.org/html/rfc7807" target="_blank">problem details</a> API responses for Aphiria applications.  This renderer tries to create a response using the following steps:
  
1. If a [custom mapping](#custom-problem-details-mappings) exists for the thrown exception, it's used to create a problem details response
2. If no mapping exists, a default 500 problem details response will be returned

By default, when the <a href="https://tools.ietf.org/html/rfc7807#section-3.1" target="_blank">type</a> field is `null`, it is automatically populated with a link to the <a href="https://tools.ietf.org/html/rfc7231#section-6" target="_blank">HTTP status code</a> contained in the problem details.  You can override this behavior by extending `ProblemDetailsExceptionRenderer` and implementing your own `getTypeFromException()`.  Similarly, the exception message is used as the title of the problem details.  If you'd like to customize that, implement your own `getTitleFromException()`.

<h3 id="custom-problem-details-mappings">Custom Problem Details Mappings</h3>

You might not want all exceptions to result in a 500.  For example, if you have a `UserNotFoundException`, you might want to map that to a 404.  Here's how:

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ProblemDetailsExceptionRenderer;
use Aphiria\Net\Http\HttpStatusCode;

$exceptionRenderer = new ProblemDetailsExceptionRenderer();
$exceptionRenderer->mapExceptionToProblemDetails(UserNotFoundException::class, status: HttpStatusCode::NotFound);
$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
$globalExceptionHandler->registerWithPhp();
```

You can also specify other properties in the problem details:

```php
$exceptionRenderer->mapExceptionToProblemDetails(
    OverdrawnException::class,
    type: 'https://example.com/errors/overdrawn',
    title: 'This account is overdrawn',
    detail: fn($ex) => "Account {$ex->accountId} is overdrawn by {$ex->overdrawnAmount}",
    status: HttpStatusCode::BadRequest,
    instance: fn($ex) => "https://example.com/accounts/{$ex->accountId}/errors/{$ex->id}",
    extensions: fn($ex) => ['overdrawnAmount' => $ex->overdrawnAmount]
);
```

> **Note:** All parameters that accept closures with the thrown exception can also take hard-coded values.

When a `ProblemDetails` instance is serialized in a response, all of its extensions are serialized as top-level properties - not as key-value pairs under an `extensions` property.

<h2 id="console-exception-renderer">Console Exception Renderer</h2>

<div class="context-framework" markdown="1">

> **Note:** Please refer to the [configuration documentation](configuration.md#component-exception-handler) to learn more about configuring exceptions in console apps.

</div>

`ConsoleExceptionRenderer` renders exceptions for Aphiria console applications.  To render the exception, it goes through the following steps:

1. If an [output writer](#output-writers) is registered for the thrown exception, it's used
2. Otherwise, the exception message and stack trace is output to the console

<h3 id="output-writers">Output Writers</h3>

Output writers allow you to write errors to the output and return a status code.

```php
use Aphiria\Console\Output\IOutput;
use Aphiria\Console\StatusCodes;
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Console\Exceptions\ConsoleExceptionRenderer;

$exceptionRenderer = new ConsoleExceptionRenderer();
$exceptionRenderer->registerOutputWriter(
    DatabaseNotFound::class,
    function (DatabaseNotFound $ex, IOutput $output) {
        $output->writeln('<fatal>Contact a sysadmin</fatal>');

        return StatusCodes::FATAL;
    }
);

// You can also register many exceptions-to-output writers
$exceptionRenderer->registerManyOutputWriters([
    DatabaseNotFound::class => function (DatabaseNotFound $ex, IOutput $output) {
        $output->writeln('<fatal>Contact a sysadmin</fatal>');

        return StatusCodes::FATAL;
    },
    // ...
]);

$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
$globalExceptionHandler->registerWithPhp();
```

<h2 id="logging">Logging</h2>

<div class="context-framework" markdown="1">

> **Note:** You can configure the PSR-3 logger used in the global exception handler by editing the `aphiria.logging` values in _config.php_.

</div>
<div class="context-library" markdown="1">

To configure your logger, you must add a handler:

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;
use Psr\Log\LogLevel;

$exceptionRenderer = new ApiExceptionRenderer();
$logger = new Logger('app');
$logger->pushHandler(new StreamHandler('/etc/logs/errors.txt', LogLevel::DEBUG));
$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer, $logger);
$globalExceptionHandler->registerWithPhp();
```

</div>

<h3 id="exception-log-levels">Exception Log Levels</h3>

It's possible to map certain exceptions to a PSR-3 log level.  For example, if you have an exception that means your infrastructure might be down, you can cause it to log as an emergency.

<div class="context-framework" markdown="1">

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\Schemes\CookieAuthenticationOptions;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withLogLevelFactory(
            $appBuilder,
            DatabaseNotFoundException::class,
            fn(DatabaseNotFoundException $ex) => LogLevel::EMERGENCY
        );
    }
}
```

</div>

<div class="context-library" markdown="1">

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;
use Psr\Log\LogLevel;

$globalExceptionHandler = new GlobalExceptionHandler(new ApiExceptionRenderer());
$globalExceptionHandler->registerLogLevelFactory(
    DatabaseNotFoundException::class,
    fn(DatabaseNotFoundException $ex) => LogLevel::EMERGENCY
);

// You can also register multiple exceptions-to-log-level factories
$globalExceptionHandler->registerManyLogLevelFactories([
    DatabaseNotFoundException::class => fn(DatabaseNotFoundException $ex) => LogLevel::EMERGENCY,
    // ...
]);
```

</div>
