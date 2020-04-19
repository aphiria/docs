<h1 id="doc-title">Exception Handling</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Global Exception Handler](#global-exception-handler)
2. [API Exception Renderer](#api-exception-renderer)
   1. [Exception Responses](#exception-responses)
3. [Console Exception Renderer](#console-exception-renderer)
   1. [Output Writers](#output-writers)
3. [Logging](#logging)
   1. [Exception Log Levels](#exception-log-levels)

</div>

</nav>

<h2 id="global-exception-handler">Global Exception Handler</h2>

Sometimes, your application is going to throw an unhandled exception or shut down unexpectedly.  When this happens, it would be nice to log details about the error and present a nicely-formatted response for the user.  Aphiria provides `GlobalExceptionHandler` to do just this.  It can be used to render exceptions for both HTTP and console applications, and is framework-agnostic.

Let's look at an example:

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;

$exceptionRenderer = new ApiExceptionRenderer();
$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
// This is important - it's what registers the handler as the default handler in PHP
$globalExceptionHandler->registerWithPhp();
```

That's it.  Now, whenever an unhandled error or exception is thrown, the global exception handler will catch it, [log it](#logging), and [render it](#http-exception-renderer).  We'll go into more details on how to customize it below.

<h2 id="api-exception-renderer">API Exception Renderer</h2>

`ApiExceptionRenderer` is provided out of the box to simplify rendering API responses for Aphiria applications.  This renderer tries to create a response using the following steps:
  
1. If an [exception response](#exception-responses) exists for the thrown exception, it's used
2. Otherwise, if the renderer is configured to use <a href="https://tools.ietf.org/html/rfc7807" target="_blank">problem details</a>, it will create a 500 response with a problem details body
3. Otherwise, if the renderer does not use problem details, an empty 500 response is used

To turn problem detail responses off, you pass in `false` in the constructor:

```php
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;

$exceptionRenderer = new ApiExceptionRenderer(false);
```

<h3 id="exception-responses">Exception Responses</h3>

You might not want all exceptions to result in a 500.  For example, if you have a `UserNotFoundException`, you might want to map that to a 404.  Here's how:

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;
use Aphiria\Net\Http\HttpStatusCodes;
use Aphiria\Net\Http\IHttpRequestMessage;
use Aphiria\Net\Http\IResponseFactory;

$exceptionRenderer = new ApiExceptionRenderer();
$exceptionRenderer->registerResponseFactory(
    UserNotFoundException::class,
    function (UserNotFoundException $ex, IHttpRequestMessage $request, IResponseFactory $responseFactory) {
        return $responseFactory->createResponse($request, HttpStatusCodes::HTTP_NOT_FOUND);
    }
);

// You can also register many exceptions-to-response factories:
$exceptionRenderer->registerManyResponseFactories([
    UserNotFoundException::class => 
        function (UserNotFoundException $ex, IHttpRequestMessage $request, IResponseFactory $responseFactory) {
            return $responseFactory->createResponse($request, HttpStatusCodes::HTTP_NOT_FOUND);
        },
    // ...
]);

$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
$globalExceptionHandler->registerWithPhp();
```

<h2 id="console-exception-renderer">Console Exception Renderer</h2>

`ConsoleExceptionRenderer` renders exceptions for Aphiria console applications.  To render the exception, it goes through the following steps:

1. If an [output writer](#output-writers) is registered for the thrown exception, it's used
2. Otherwise, the exception message and stack trace is output to the console

<h3 id="output-writers">Output Writers</h3>

Output writers allow you to write errors to the output and return a status code.

```php
use Aphiria\Console\Output\IOutput;
use Aphiria\Console\StatusCodes;
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Exceptions\Console\ConsoleExceptionRenderer;

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

By default, the global exception handler is compatible with any PSR-3 logger such as <a href="https://github.com/Seldaek/monolog" target="_blank">Monolog</a>.  To use a specific logger, just pass it into the handler:

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

<h3 id="exception-log-levels">Exception Log Levels</h3>

It's possible to map certain exceptions to a PSR-3 log level.  For example, if you have an exception that means your infrastructure might be down, you can cause it to log as an emergency.

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;
use Psr\Log\LogLevel;

$globalExceptionHandler = new GlobalExceptionHandler(new ApiExceptionRenderer());
$globalExceptionHandler->registerLogLevelFactory(
    DatabaseNotFoundException::class,
    fn (DatabaseNotFoundException $ex) => LogLevel::EMERGENCY
);

// You can also register multiple exceptions-to-log-level factories
$globalExceptionHandler->registerManyLogLevelFactories([
    DatabaseNotFoundException::class => fn (DatabaseNotFoundException $ex) => LogLevel::EMERGENCY,
    // ...
]);
```
