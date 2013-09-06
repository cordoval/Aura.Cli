# Aura.Cli

## Overview

The Aura.Cli library provides the equivalent of request ( _Context_ ) and
response ( _Stdio_ ) objects for the command line interface, including
_Getopt_ support. Note that it does not provide commands or other
controller-like objects; it is strictly for environment discovery and standard
input/ouput operations.

### Installation and Autoloading

This library is installable via Composer and is registered on Packagist at
<https://packagist.org/packages/aura/cli>. Installing via Composer will set up
autoloading automatically.

Alternatively, download or clone this repository, then require or include its
_autoload.php_ file.

### Dependencies

As with all Aura libraries, this library has no external dependencies.

### Tests

[![Build Status](https://travis-ci.org/auraphp/Aura.Cli.png)](https://travis-ci.org/auraphp/Aura.Cli)

This library has 100% code coverage. To run the library tests, first install
[PHPUnit][], then go to the library _tests_ directory and issue `phpunit` at
the command line.

[PHPUnit]: http://phpunit.de/manual/

### API Documentation

This library has embedded DocBlock API documentation. To generate the
documentation in HTML, first install [PHPDocumentor][] or [ApiGen][], then go
to the library directory and issue one of the following at the command line:

    # for PHPDocumentor
    phpdoc -d ./src/ -t /path/to/output/
    
    # for ApiGen
    apigen --source=./src/ --destination=/path/to/output/

You can then browse the HTML-formatted API documentation at _/path/to/output_.

[PHPDocumentor]: http://phpdoc.org/docs/latest/for-users/installation.html
[ApiGen]: http://apigen.org/#installation

### PSR Compliance

This library is compliant with [PSR-1][] and [PSR-2][]. If you notice
compliance oversights, please send a patch via pull request.

[PSR-1]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md
[PSR-2]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md


## Usage

The two main objects _Context_ and _Stdio_.

### The Context Object

The _Context_ object provides information about the command line environment,
including any option flags passed via the command line. (This is the command
line equivalent of a web request object.)

Instantiate a _Context_ object using the _CliFactory_; pass it a copy of
`$GLOBALS`.

```php
<?php
use Aura\Cli\CliFactory;

$cli_factory = new CliFactory;
$context = $cli_factory->newContext($GLOBALS);
?>
```

You can access the `$_ENV`, `$_SERVER`, and `$argv` values with the `$env`,
`$server`, and `$argv` property objects, respectively. (Note that these
properties are copies of those superglobals as they were are at the time of
_Context_ instantiation.) You can pass an alternative default value if the
related key is missing.

```php
<?php
// get copies of superglobals
$env    = $context->env->get();
$server = $context->server->get();
$argv   = $context->argv->get();

// equivalent to:
// $value = isset($_ENV['key']) ? $_ENV['key'] : null;
$value = $context->env->get('key');

// equivalent to:
// $value = isset($_ENV['key']) ? $_ENV['key'] : 'other_value';
$value = $context->env->get('key', 'other_value');
?>
```

### Getopt Support

The _Context_ object provides support for retrieving command-line options and
params, along with positional and named arguments.

To retrieve options and arguments parsed from the command-line `$argv` values,
use the `getopt()` method on the _Context_ object. This will return a
_GetoptValues_ object for you to use as as you wish.

#### Defining Options and Params

To tell `getopt()` how to recognize command line options, pass an array of
option definitions. The definitions array format is similar to, but not
exactly the same as, the one used by the [getopt()](http://php.net/getopt)
function in PHP. Instead of defining short flags in a string and long options
in a separate array, they are both defined as elements in a single array.

```php
<?php
$opt_defs = [
    'a',     // short flag -a, parameter is not allowed
    'b:',    // short flag -b, parameter is required
    'c::',   // short flag -c, parameter is optional
    'foo',   // long option --foo, parameter is not allowed
    'bar:',  // long option --bar, parameter is required
    'baz::', // long option --baz, parameter is optional
];

$getopt = $context->getopt($opt_defs);
?>
```

Then use the `get()` method on the returned _GetoptValues_ object to retrieve
the option values; you can provide an alternative default value for when the
option is missing.

```php
<?php
$a   = $getopt->get('-a', 0); // 1 if -a was passed, 0 if not
$b   = $getopt->get('-b');
$c   = $getopt->get('-c', 'default value');
$foo = $getopt->get('--foo', 0); // 1 if --foo was passed, 0 if not
$bar = $getopt->get('--bar');
$baz = $getopt->get('--baz', 'default value');
?>
```

If you want a short flag and a long option to be mapped to the same long
option name, pass the short flag as an array key and the long option name as
its value:

```php
<?php
// map -f to --foo
$opt_defs = [
    'f:' => 'foo',  // short flag -f, parameter required
    'foo:',         // long option --foo, parameter required
];

$getopt = $context->getopt($opt_defs);

$foo = $getopt->get('--foo'); // both -f and --foo map to '--foo'
?>
```

If an option is passed multiple times, it will result in an array of multiple
values. Options that do not take parameters will result in a count of how many
times the option was passed.

```php
<?php
$opt_defs = [
    'f',
    'foo:'
];

$getopt = $context->getopt($opt_defs);

// if the script was invoked with:
// php script.php --foo=foo --foo=bar --foo=baz -f -f -f
$foo = $getopt->get('--foo'); // ['foo', 'bar', 'baz']
$f   = $getopt->get('-f'); // 3
?>
```

If the user passes options that do not conform to the definitions, the
_GetoptValues_ object retains various errors related to the parsing
failures. In these cases, `hasErrors()` will return `true`, and you can then
review the errors.  (The errors are actually `Aura\Cli\Exception` objects,
but they don't get thrown as they occur; this is so that you can deal with or
ignore the different kinds of errors as you like.)

```php
<?php
$getopt = $context->getopt($opt_defs);
if ($getopt->hasErrors()) {
    $errors = $getopt->getErrors();
    foreach ($errors as $error) {
        // print error messages to stderr using a Stdio object
        $stdio->errln($error->getMessage());
    }
};
?>
```

#### Positional Arguments

To get the positional arguments passed to the command line, use the `get()`
method and the argument position number:

```php
<?php
$getopt = $context->getopt();

// if the script was invoked with:
// php script.php arg1 arg2 arg3 arg4

$val0 = $getopt->get(0); // script.php
$val1 = $getopt->get(1); // arg1
$val2 = $getopt->get(2); // arg2
$val3 = $getopt->get(3); // arg3
$val4 = $getopt->get(4); // arg4
?>
```

Defined options will be removed from the arguments automatically.

```php
<?php
$opt_defs = [
    'a',
    'foo:',
];

$getopt = $context->getopt($opt_defs);

// if the script was invoked with:
// php script.php arg1 --foo=bar -a arg2
$arg0 = $getopt->get(0); // script.php
$arg1 = $getopt->get(1); // arg1
$arg2 = $getopt->get(2); // arg2
$foo  = $getopt->get('--foo'); // bar
$a    = $getopt->get('-a'); // 1
?>
```

> N.b.: If an short flag has an optional parameter, the argument immediately
> after it will it will be treated as the option value, not as an argument.


#### Named Arguments

To set names on positional arguments, pass a second array to `getopt()` where
the key is the argument position and the value is the argument name you would
like to use. (The positional arguments will also be retained.)

```php
<?php
// the option definitions
$opt_defs = [
    'a',
    'foo:',
];

// the names for argument positions
$arg_defs = [
    0 => 'script_name',
    1 => 'first_arg',
    2 => 'second_arg',
];

$getopt = $context->getopt($opt_defs, $arg_defs);

// if the script was invoked with:
// php script.php arg1 --foo=bar -a arg2
$arg0        = $getopt->get(0);             // script.php
$script_name = $getopt->get('script_name'); // script.php
$arg1        = $getopt->get(1);             // arg1
$first_arg   = $getopt->get('first_arg');   // arg1
$arg2        = $getopt->get(2);             // arg2
$second_arg  = $getopt->get('second_arg');  // arg2
$foo         = $getopt->get('--foo');       // bar
$a           = $getopt->get('-a');          // 1
?>
```


### The Stdio Object

The _Stdio_ object to allows you to work with standard input/output streams.
(This is the command line equivalent of a web response object.)

Instantiate a _Stdio_ object using the _CliFactory_.

```php
<?php
use Aura\Cli\CliFactory;

$cli_factory = new CliFactory;
$stdio = $cli_factory->newStdio();
?>
```

It defaults to using `php://stdin`, `php://stdout`, and `php://stderr`, but
you can pass whatever stream names you like as parameters to the `newStdio()`
method.

The _Stdio_ object methods are ...

- `getStdin()`, `getStdout()`, and `getStderr()` to return the respective
  _Handle_ objects;

- `outln()` and `out()` to print to _stdout_, with or without a line ending;

- `errln()` and `err()` to print to _stderr_, with or without a line ending;

- `inln()` and `in()` to read from _stdin_ until the user hits enter; `inln()`
  leaves the trailing line ending in place, whereas `in()` strips it.

You can use special formatting markup in the output and error strings to set
text color, text weight, background color, and other display characteristics.
See the [formatter cheat sheet](#formatter-cheat-sheet) below.

```php
<?php
// print to stdout
$stdio->outln('This is normal text.');

// print to stderr
$stdio->errln('<<red>>This is an error in red.');
$stdio->errln('Output will stay red until a formatting change.<<reset>>');
?>
```


### Exit Codes

This library comes with a _Status_ class that defines constants for exit
status codes. You should use these whenever possible.  For example, if a
command is used with the wrong number of arguments or improper option flags,
`exit()` with `Status::USAGE`.  The exit status codes are the same as those
found in [sysexits.h](http://www.unix.com/man-page/freebsd/3/sysexits/).


### Writing Commands

The Aura.Cli library does not come with an abstract or base command class to
extend from, but writing commands for yourself is straightforward. The
following is a standalone command script, but similar logic can be used in a
class.  Save it in a file named `hello` and invoke it with
`php hello [-v,--verbose] [name]`.

```php
<?php
use Aura\Cli\CliFactory;
use Aura\Cli\Status;

require '/path/to/Aura.Cli/autoload.php';

// get the context and stdio objects
$cli_factory = new CliFactory;
$context = $cli_factory->newContext($GLOBALS);
$stdio = $cli_factory->newStdio();

// define options and named arguments through getopt
$opt_defs = ['v' => 'verbose', 'verbose'];
$arg_defs = [1 => 'name'];
$getopt = $context->getopt($opt_defs, $arg_defs);

// do we have a name to say hello to?
$name = $getopt->get('name');
if (! $name) {
    // print an error
    $stdio->errln("Please give a name to say hello to.");
    exit(Status::USAGE);
}

// say hello
if ($getopt->get('--verbose')) {
    // verbose output
    $stdio->outln("Hello {$name}, it's nice to see you!");
} else {
    // plain output
    $stdio->outln("Hello {$name}!");
}

// done!
exit(Status::SUCCESS);
?>
```


### Formatter Cheat Sheet

On POSIX terminals, `<<markup>>` strings will change the display
characteristics. Note that these are not HTML tags; they will be converted
into terminal control codes, and do not get "closed". You can place as many
space-separated markup codes between the double angle-brackets as you like.

    reset       reset display to defaults
    
    black       black text
    red         red text
    green       green text
    yellow      yellow text
    blue        blue text
    magenta     magenta (purple) text
    cyan        cyan (light blue) text
    white       white text
    
    blackbg     black background
    redbg       red background
    greenbg     green background
    yellowbg    yellow background
    bluebg      blue background
    magentabg   magenta (purple) background
    cyanbg      cyan (light blue) background
    whitebg     white background

    bold        bold in the current text and background colors
    dim         dim in the current text and background colors
    ul          underline in the current text and background colors
    blink       blinking in the current text and background colors
    reverse     reverse the current text and background colors

For example, to set bold white text on a red background, add `<<bold white redbg>>`
into your output or error string. Reset back to normal with `<<reset>>`.
