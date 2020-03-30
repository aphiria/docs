<h1 id="doc-title">Sessions</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
2. [Basic Usage](#basic-usage)
   1. [Setting Data](#setting-data)
   2. [Getting Data](#getting-data)
   3. [Getting All Data](#getting-all-data)
   4. [Checking if a Session Has a Key](#checking-if-session-has-key)
   5. [Deleting Data](#deleting-data)
   6. [Flushing All Data](#flushing-all-data)
   7. [Flashing Data](#flashing-data)
   8. [Regenerating the Id](#regenerating-the-id)
3. [Session Handlers](#session-handlers)
4. [Using Sessions In Controllers](#using-sessions-in-controllers)
5. [Middleware](#middleware)
6. [ID Generators](#id-generators)
7. [Encrypting Session Data](#encrypting-session-data)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

HTTP is a stateless protocol.  What that means is that each request has no memory of previous requests.  If you've ever used the web, though, you've probably noticed that websites are able to remember information across requests.  For example, a "shopping cart" on an e-commerce website remembers what items you've added to your cart.  How'd they do that?  Sessions.

> **Note:** Although similar in concept, Aphiria's sessions do not use PHP's built-in `$_SESSION` functionality because it is awful.

<h2 id="basic-usage">Basic Usage</h2>

Aphiria sessions must implement `ISession` (`Session` comes built-in).

<h4 id="setting-data">Setting Data</h4>

Any kind of serializable data can be written to sessions:

```php
use Aphiria\Sessions\Session;

$session = new Session();
$session->set('someString', 'foo');
$session->set('someArray', ['bar', 'baz']);
```

<h4 id="getting-data">Getting Data</h4>

```php
$session->set('someKey', 'myValue');
echo $session->get('someKey'); // "myValue"
```

<h4 id="getting-all-data">Getting All Data</h4>

```php
$session->set('foo', 'bar');
$session->set('baz', 'blah');
$data = $session->getAll();
echo $data[0]; // "bar"
echo $data[1]; // "blah"
```

<h4 id="checking-if-session-has-key">Checking if a Session Has a Key</h4>

```php
echo $session->containsKey('foo'); // 0
$session->set('foo', 'bar');
echo $session->containsKey('foo'); // 1
```

<h4 id="deleting-data">Deleting Data</h4>

```php
$session->delete('someKey');
```

<h4 id="flushing-all-data">Flushing All Data</h4>

```php
$session->flush();
```

<h4 id="flashing-data">Flashing Data</h4>

Let's say you're writing a form that can display any validation errors after submitting, and you'd like to remember these error messages only for the next request.  Use `flash()`:

```php
$session->flash('formErrors', ['Username is required', 'Invalid email address']);
// ...redirect back to the form...
foreach ($session->get('formErrors') as $error) {
    echo htmlentities($error);
}
```

On the next request, the data in "formErrors" will be deleted.  Want to extend the lifetime of the flash data by one more request?  Use `reflash()`.

<h4 id="regenerating-the-id">Regenerating the Id</h4>

```php
$session->regenerateId();
```

<h2 id="session-handlers">Session Handlers</h2>

Session handlers are what actually read and write session data from some form of storage, eg text files, cache, or cookies, and are typically invoked in [middleware](#middleware).  All Aphiria handlers implement `\SessionHandlerInterface` (built-into PHP).  Aphiria has the concept of session "drivers", which represent the storage that powers the handlers.  For example, `FileSessionDriver` stores session data to plain-text files, and `ArraySessionDriver` writes to an in-memory array, which can be useful for development environments.  Aphiria contains a session handler already set up to use a driver:

```php
use Aphiria\Sessions\Handlers\FileSessionDriver;
use Aphiria\Sessions\Handlers\DriverSessionHandler;

$driver = new FileSessionDriver('/tmp/sessions');
$handler = new DriverSessionHandler($driver);
```

<h2 id="using-sessions-in-controllers">Using Sessions in Controllers</h2>

To use sessions in your controllers, simply inject it into the controller's constructor along with a type hint:

```php
namespace App\Authentication\Api\Controllers;

use Aphiria\Sessions\ISession;

class UserController extends Controller
{
    private ISession $session;

    // The session will be automatically injected into the controller by the router
    public function __construct(ISession $session)
    {
        $this->session = $session;
    }

    public function logIn(LoginDto $loginDto): IHttpResponseMessage
    {
        // Do the login...

        $this->session->set('user', (string)$user);
 
        // Create the response...
    }
}
```

<h2 id="middleware">Middleware</h2>

Middleware is the best way to handle reading session data from storage and persisting it back to storage at the end of a request.  For convenience, Aphiria provides `Aphiria\Sessions\Middleware\Session` to handle reading and writing session data to cookies.  Let's look at how to configure the middleware:

```php
use Aphiria\Sessions\Middleware\Session as SessionMiddleware;

// Assume our session and handler are already created...
$sessionMiddleware = new SessionMiddleware(
    $session,
    $sessionHandler,
    3600,        // Session TTL in seconds
    'sessionid', // The name of the cookie
    '/'          // The path the cookie is valid for
);
```

Refer to the [application builder library](application-builders.md#component-middleware) for more information on how to register the middleware.

<h2 id="id-generators">ID Generators</h2>

If your session has just started or if its data has been invalidated, a new session ID will need to be generated.  These Ids must be cryptographically secure to prevent session hijacking.  If you're using `Session`, you can either pass in your own ID generator (must implement `IIdGenerator`) or use the default `UuidV4IdGenerator`.

> **Note:** It's recommended you use Aphiria's `UuidV4IdGenerator` unless you know what you're doing.

<h2 id="encrypting-session-data">Encrypting Session Data</h2>

You might find yourself storing sensitive data in sessions, in which case you'll want to encrypt it.  To do this, pass in an instance of `ISessionEncrypter` to `DriverSessionHandler` (passing in `null` will cause your data to be unencrypted).

```php
use Aphiria\Sessions\Handlers\DriverSessionHandler;
use Aphiria\Sessions\Handlers\ISessionEncrypter;

$driver = new FileSessionDriver('/tmp/sessions');
$encrypter = new class () implements ISessionEncrypter {
    // Implement ISessionEncrypter...
};
$handler = new DriverSessionHandler($driver, $encrypter);
```

Now, all your session data will be encrypted before being written and decrypted after being read.
