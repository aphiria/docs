<h1 id="doc-title">Console</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Running Commands](#running-commands)
   1. [Getting Help](#getting-help)
3. [Creating Commands](#creating-commands)
   1. [Registering Commands](#registering-commands)
   2. [Arguments](#arguments)
   3. [Options](#options)
   4. [Calling From Code](#calling-from-code)
4. [Command Annotations](#command-annotations)
    1. [Example](#command-annotation-example)
    4. [Scanning For Annotations](#scanning-for-annotations)
5. [Prompts](#prompts)
   1. [Confirmation](#confirmation)
   2. [Multiple Choice](#multiple-choice)
6. [Output](#output)
7. [Formatters](#formatters)
   1. [Padding](#padding)
   2. [Tables](#tables)
   3. [Progress Bars](#progress-bars)
8. [Style Elements](#style-elements)
   1. [Built-In Elements](#built-in-elements)
   2. [Custom Elements](#custom-elements)
   3. [Overriding Built-In Elements](#overriding-built-in-elements)

</div>

</nav>
  
<h2 id="basics">Basics</h2>

Console applications are great for administrative tasks and code generation.  With Aphiria, you can easily create your own console commands, display question prompts, and use HTML-like syntax for output styling.

If you're already using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, you can skip to the [next section](#running-commands).  Otherwise, let's create a file called _aphiria_ and paste the following code into it:

```php
#!/usr/bin/env php
<?php

use Aphiria\Console\App;
use Aphiria\Console\Commands\CommandRegistry;

require_once __DIR__ . '/vendor/autoload.php';

$commands = new CommandRegistry();

// Register your commands here...

global $argv;
exit((new App($commands))->handle($argv));
```

Now, you're set to start [running commands](#running-commands).

<h2 id="running-commands">Running Commands</h2>

To run commands, type `php aphiria COMMAND_NAME` into a terminal from the directory that Aphiria is installed in.

<h3 id="getting-help">Getting Help</h3>

To get help with any command, use the help command:

```bash
php aphiria help COMMAND_NAME
```

<h2 id="creating-commands">Creating Commands</h2>

In Aphiria, a command defines the name, [arguments](#arguments), and [options](#options) that make up a command.  Each command has a command handler, which is what actually processes a command.

Let's take a look at an example:

```php
use Aphiria\Console\Commands\Command;
use Aphiria\Console\Input\{Argument, ArgumentTypes, Input, Option, OptionTypes};
use Aphiria\Console\Output\IOutput;

$greetingCommand = new Command(
    'greet',
    [new Argument('name', ArgumentTypes::REQUIRED, 'The name to greet')],
    [new Option('yell', 'y', OptionTypes::OPTIONAL_VALUE, 'Yell the greeting?', 'yes')],
    'Greets a person'
);
$greetingCommandHandler = function (Input $input, IOutput $output) {
    $greeting = "Hello, {$input->arguments['name']}";

    if ($input->options['yell'] === 'yes') {
        $greeting = strtoupper($greeting);
    }

    $output->writeln($greeting);
};
```

Your command handler can either be a `Closure` that takes the input and output as parameters, or it can implement `ICommandHandler`, which has a single method `handle()` that accepts the same parameters.  If you pass in a `Closure`, it will be wrapped in a `ClosureCommandHandler`.

The following properties are available to you in `Input`:

```php
$input->commandName; // The name of the command that was invoked
$input->arguments['argName']; // The value of 'argName'
$input->options['optionName']; // The value of 'optionName'
```

If you're checking to see if an option that does not have a value is set, use `array_key_exists('optionName', $input->options)` - the value will be `null`, and `isset()` will return `false`.

> **Note:** `$input->options` stores option values by their long names.  Do not try to access them by their short names.

<h3 id="registering-commands">Registering Commands</h3>

Before you can use the example command, you must register it so that the `App` knows about it.  Your command handler should be wrapped in a parameterless closure that will return the handler.  This allows us to defer resolving a handler until we actually need it.  This is especially useful when your handler is a class with expensive-to-instantiate dependencies, such as database connections.

> **Note:** If you're using the configuration library, refer to [its documentation](application-builders.md#configuring-console-commands) to learn how to register your commands to your app.

```php
use Aphiria\Console\App;
use Aphiria\Console\Commands\CommandRegistry;

$commands = new CommandRegistry();
$commands->registerCommand(
    $greetingCommand, 
    function () use ($greetingCommandHandler) {
        return $greetingCommandHandler;
    }
);

// Actually run the application
global $argv;
exit((new App($commands))->handle($argv));
```

To call this command, run this from the command line:

```bash
php aphiria greet Dave -y
```

This will output:

```
HELLO, DAVE
```

>> **Note:** Aphiria uses `opis/closure` to serialize and deserialize closures.  At this time, it does not support short closures.  So, any manually registered command handler factories **must** use long-form closures.

<h3 id="arguments">Arguments</h3>

Console commands can accept arguments from the user.  Arguments can be required, optional, and/or arrays.  You specify the type by bitwise OR-ing the different arguments types.  Array arguments allow a variable number of arguments to be passed in, like "php aphiria foo arg1 arg2 arg3 ...".  The only catch is that array arguments must be the last argument defined for the command.

Let's take a look at an example argument:

```php
use Aphiria\Console\Input\Argument;
use Aphiria\Console\Input\ArgumentTypes;

// The argument will be required and an array
$type = ArgumentTypes::REQUIRED | ArgumentTypes::IS_ARRAY;
// The description argument is used by the help command
$argument = new Argument('foo', $type, 'The foo argument');
```

>**Note:** Like array arguments, optional arguments must appear after any required arguments.

<h3 id="options">Options</h3>

You might want different behavior in your command depending on whether or not an option is set.  This is possible using `Aphiria\Console\Input\Option`.  Options have two formats:

1. Short, eg "-h"
2. Long, eg "--help"

<h4 id="short-names">Short Names</h4>

Short option names are always a single letter.  Multiple short options can be grouped together.  For example, `-rf` means that options with short codes "r" and "f" have been specified.  The default value will be used for short options.

<h4 id="long-names">Long Names</h4>

Long option names can specify values in two ways:  `--foo=bar` or `--foo bar`.  If you only specify `--foo` for an optional-value option, then the default value will be used.

<h4 id="array-options">Array Options</h4>

Options can be arrays, eg `--foo=bar --foo=baz` will set the "foo" option to `["bar", "baz"]`.

Like arguments, option types can be specified by bitwise OR-ing types together.  Let's look at an example:

```php
use Aphiria\Console\Input\Option;
use Aphiria\Console\Input\OptionTypes;

$type = OptionTypes::IS_ARRAY | OptionTypes::REQUIRED_VALUE;
$option = new Option('foo', 'f', $types, 'The foo option');
```

<h3 id="calling-from-code">Calling From Code</h3>

It's possible to call a command from another command by using `App`:

```php
use Aphiria\Console\Input\Input;
use Aphiria\Console\Output\IOutput;

$commandHandler = function (Input $input, IOutput $output) use ($app) {
    $app->handle('foo arg1 --option1=value', $output);
    
    // Do other stuff...
};

// Register your commands...
```

Alternatively, if your handler is a class, you could inject the app via the constructor:

```php
use Aphiria\Console\Commands\ICommandBus;
use Aphiria\Console\Commands\ICommandHandler;
use Aphiria\Console\Input\Input;
use Aphiria\Console\Output\IOutput;

final class FooCommandHandler implements ICommandHandler
{
    private ICommandBus $app;
    
    public function __construct(ICommandBus $app)
    {
        $this->app = $app;
    }
    
    public function handle(Input $input, IOutput $output)
    {
        $this->app->handle('foo arg1 --option1=value', $output);
    
        // Do other stuff...
    };
}
```

If you want to call the other command but not write its output, use the `Aphiria\Console\Output\SilentOutput` output.

> **Note:** If a command is being called by a lot of other commands, it might be best to refactor its actions into a separate class.  This way, it can be used by multiple commands without the extra overhead of calling console commands through PHP code.

<h2 id="command-annotations">Command Annotations</h2>

Sometimes, it's convenient to define your command alongside your command handler so you don't have to jump back and forth remembering what arguments or options your command takes.  Aphiria offers the option to do so via annotations.

<h3 id="command-annotation-example">Command Annotation Example</h3>

Let's look at an example that duplicates the [greeting example from above](#registering-commands):

```php
/**
 * @Command(
 *     "greet", 
 *     arguments={@Argument("name", type=ArgumentTypes::REQUIRED, description="The name to greet")},
 *     options={@Option("yell", shortName="y", type=OptionTypes::OPTIONAL_VALUE, description="Yell the greeting?", defaultValue="yes")},
 *     description="Greets a person"
 * )
 */
final class GreetingCommandHandler implements ICommandHandler
{
    public function handle(Input $input, IOutput $output)
    {
        // ...
    }
}
```

<h3 id="scanning-for-annotations">Scanning For Annotations</h3>

Before you can use annotations, you'll need to configure Aphiria to scan for them.  The [configuration](application-builders.md) library provides a convenience method for this:

```php
use Aphiria\Configuration\AphiriaComponentBuilder;
use Aphiria\ConsoleCommandAnnotations\ICommandAnnotationRegistrant;
use Aphiria\ConsoleCommandAnnotations\ReflectionCommandAnnotationRegistrant;

// Assume we already have $container set up
$commandAnnotationRegistrant = new ReflectionCommandAnnotationRegistrant(['PATH_TO_SCAN']);
$container->bindInstance(ICommandAnnotationRegistrant::class, $commandAnnotationRegistrant);

(new AphiriaComponentBuilder($container))
    ->withConsoleCommandAnnotations($appBuilder);
```

If you're not using the configuration library, you can manually configure your app to scan for annotations:

```php
use Aphiria\ConsoleCommandAnnotations\ReflectionCommandAnnotationRegistrant;
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Console\Commands\LazyCommandRegistryFactory;

$commandFactory = new LazyCommandRegistryFactory(function (CommandRegstry $commands) {
    $commandAnnotationRegistrant = new ReflectionCommandAnnotationRegistrant(['PATH_TO_SCAN']);
    $commandAnnotationRegistrant->registerCommands($commands);
});
$commands = $commandFactory->createCommands();
````

<h2 id="prompts">Prompts</h2>

Prompts are great for asking users for input beyond what is accepted by arguments.  For example, you might want to confirm with a user before doing an administrative task, or you might ask her to select from a list of possible choices.  Prompts accept `Aphiria\Console\Output\Prompts\Question` objects.

<h3 id="confirmation">Confirmation</h3>

To ask a user to confirm an action with a simple "y" or "yes", use an `Aphiria\Console\Output\Prompts\Confirmation`:

```php
use Aphiria\Console\Output\Prompts\Confirmation;
use Aphiria\Console\Output\Prompts\Prompt;

$prompt = new Prompt();
// This will return true if the answer began with "y" or "Y"
$prompt->ask(new Confirmation('Are you sure you want to continue?'), $output);
```

<h3 id="multiple-choice">Multiple Choice</h3>

Multiple choice questions are great for listing choices that might otherwise be difficult for a user to remember.  An `Aphiria\Console\Output\Prompts\MultipleChoice` accepts question text and a list of choices:

```php
use Aphiria\Console\Output\Prompts\MultipleChoice;

$choices = ['Boeing 747', 'Boeing 757', 'Boeing 787'];
$question = new MultipleChoice('Select your favorite airplane', $choices);
$prompt->ask($question, $output);
```

This will display:

```php
Select your favorite airplane
  1) Boeing 747
  2) Boeing 757
  3) Boeing 787
  >
```

If the `$choices` array is associative, then the keys will map to values rather than 1)...N).

<h2 id="output">Output</h2>

Outputs allow you to write messages to an end user.  The different outputs include:

1. `Aphiria\Console\Output\ConsoleOutput`
   * Used to write messages to the console
   * The output used by default
2. `Aphiria\Console\Output\SilentOutput`
   * Used when we don't want any messages to be written
   * Useful for when one command calls another

Each output offers three methods:

1. `readLine()`
   * Reads a line of input
1. `write()`
   * Writes a message to the existing line
2. `writeln()`
   * Writes a message to a new line
3. `clear()`
   * Clears the current screen
   * Only works in `ConsoleOutput`

<h2 id="formatters">Formatters</h2>

Formatters are great for nicely-formatting output to the console.

<h3 id="padding">Padding</h3>

The `Aphiria\Console\Output\Formatters\PaddingFormatter` formatter allows you to create column-like output.  It accepts an array of column values.  The second parameter is a callback that will format each row's contents.  Let's look at an example:

```php
use Aphiria\Console\Output\Formatters\PaddingFormatter;

$paddingFormatter = new PaddingFormatter();
$rows = [
    ['George', 'Carlin', 'great'],
    ['Chris', 'Rock', 'good'],
    ['Jim', 'Gaffigan', 'pale']
];
$paddingFormatter->format($rows, fn ($row) => $row[0] . ' - ' . $row[1] . ' - ' . $row[2]);
```

This will return:
```
George - Carlin   - great
Chris  - Rock     - good
Jim    - Gaffigan - pale
```

There are a few useful functions for customizing the padding formatter:

* `setEolChar()`
  * Sets the end-of-line character
* `setPadAfter()`
  * Sets whether to pad before or after strings
* `setPaddingString()`
  * Sets the padding string

<h3 id="tables">Tables</h3>

ASCII tables are a great way to show tabular data in a console.  To create a table, use `Aphiria\Console\Output\Formatters\TableFormatter`:

```php
use Aphiria\Console\Output\Formatters\TableFormatter;

$table = new TableFormatter();
$rows = [
    ['Sean', 'Connery'],
    ['Pierce', 'Brosnan']
];
$table->format($rows);
```

This will return:

```
+--------+---------+
| Sean   | Connery |
| Pierce | Brosnan |
+--------+---------+
```

Headers can also be included in tables:

```php
$headers = ['First', 'Last'];
$table->format($rows, $headers);
```

This will return:

```
+--------+---------+
| First  | Last    |
+--------+---------+
| Sean   | Connery |
| Pierce | Brosnan |
+--------+---------+
```

There are a few useful functions for customizing the look of tables:

* `setCellPaddingString()`
  * Sets the cell padding string
* `setEolChar()`
  * Sets the end-of-line character
* `setHorizontalBorderChar()`
  * Sets the horizontal border character
* `setIntersectionChar()`
  * Sets the row/column intersection character
* `setPadAfter()`
  * Sets whether to pad before or after strings
* `setVerticalBorderChar()`
  * Sets the vertical border character
    
<h3 id="progress-bars">Progress Bars</h3>

Progress bars a great way to visually indicate to a user the progress of a long-running task, like this one:

```
[=================50%--------------------] 50/100
Time remaining: 15 secs
```  

Creating one is simple - you just create a `ProgressBar`, and specify the formatter to use:

```php
use Aphiria\Console\Output\Formatters\{ProgressBar, ProgressBarFormatter};

// Assume our output is already created
$formatter = new ProgressBarFormatter($output);
// Set the maximum number of "steps" to 100
$progressBar = new ProgressBar(100, $formatter);
```

You can advance your progress bar:

```php
$progressBar->advance();

// Or with a custom step

$progressBar->advance(2);
```

Alternatively, you can set a specific progress:

```php
$progressBar->setProgress(50);
```

To explicitly complete the progress bar, call

```php
$progressBar->complete();
```

Each time progress is made, the formatter will be updated, and will draw a new bar.

<h4 id="customizing-progress-bars">Customizing Progress Bars</h4>

You may customize the characters used in your progress bar via:

```php
$formatter->completedProgressChar = '*';
$formatter->remainingProgressChar = '-';
```

If you'd like to customize the format of the progress bar text, you may by specifying `sprintf()`-encoded text.  The following placeholders are built in for you to use:

* `%progress%` - The current progress
* `%maxSteps%` - The max number of steps
* `%bar%` - The actual progress bar that's drawn
* `%timeRemaining%` - The amount of time remaining
* `%percent%` - The current progress as a percentage

To specify the format, pass it into `ProgressBarFormatter`:

```php
$formatter = new ProgressBarFormatter(
    $output,
    80,
    '%bar% - Time remaining: %timeRemaining%'
);
```

<h2 id="style-elements">Style Elements</h2>

Aphiria supports HTML-like style elements to perform basic output formatting like background color, foreground color, boldening, and underlining.  For example, writing:

```
<b>Hello!</b>
```

...will output "<b>Hello!</b>".  You can even nest elements:

```
<u>Hello, <b>Dave</b></u>
```

..., which will output an underlined string where "Dave" is both bold AND underlined.

<h3 id="built-in-elements">Built-In Elements</h3>

The following elements come built-into Aphiria:
* &lt;success&gt;&lt;/success&gt;
* &lt;info&gt;&lt;/info&gt;
* &lt;question&gt;&lt;/question&gt;
* &lt;comment&gt;&lt;/comment&gt;
* &lt;error&gt;&lt;/error&gt;
* &lt;fatal&gt;&lt;/fatal&gt;
* &lt;b&gt;&lt;/b&gt;
* &lt;u&gt;&lt;/u&gt;

<h3 id="custom-elements">Custom Elements</h3>

You can create your own style elements.  Elements are registered to `Aphiria\Console\Output\Compilers\Elements\ElementRegistry`.

```php
use Aphiria\Console\Output\Compilers\Elements\{Colors, Element, ElementRegistry, Style, TextStyles};
use Aphiria\Console\Output\Compilers\OutputCompiler;
use Aphiria\Console\Output\ConsoleOutput;

$elements = new ElementRegistry();
$elements->registerElement(
    new Element('foo', new Style(Colors::BLACK, Colors::YELLOW, [TextStyles::BOLD])
);
$outputCompiler = new OutputCompiler($elements);
$output = new ConsoleOutput($outputCompiler);

// Now, pass it into the app (assume it's already set up)
global $argv;
exit($app->handle($argv, $output));
```

<h3 id="overriding-built-in-elements">Overriding Built-In Elements</h3>

To override a built-in element, just re-register it:

```php
$compiler->registerElement(
    new Element('success', new Style(Colors::GREEN, Colors::BLACK))
);
```
