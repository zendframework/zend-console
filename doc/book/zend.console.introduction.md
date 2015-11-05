# Introduction to Zend\\Console

Zend Framework 2 features built-in console support.

When a `Zend\Application` is run from a console window (a shell window or Windows command prompt),
it will recognize this fact and prepare `Zend\Mvc` components to handle the request. Console support
is enabled by default, but to function properly it requires at least one console route
&lt;zend.console.routes&gt; and one action controller &lt;zend.mvc.controllers&gt; to handle the
request.

- Console routing &lt;zend.console.routes&gt; allows you to invoke controllers and action depending
on command line parameters provided by the user.
- Module Manager integration &lt;zend.console.modules&gt; allows ZF2 applications and modules to
display help and usage information, in case the command line has not been understood (no route
matched).
- Console-aware action controllers &lt;zend.console.controllers&gt; will receive a console request
containing all named parameters and flags. They are able to send output back to the console window.
- Console adapters&lt;zend.console.adapter&gt; provide a level of abstraction for interacting with
console on different operating systems.
- Console prompts &lt;zend.console.prompts&gt; can be used to interact with the user by asking him
questions and retrieving input.

## Writing console routes

A console route defines required and optional command line parameters. When a route matches, it
behaves analogical to a standard, http route &lt;zend.mvc.routing&gt; and can point to a MVC
controller &lt;zend.mvc.controllers&gt; and an action.

Let's assume that we'd like our application to handle the following command line:

``` sourceCode
> zf user resetpassword user@mail.com
```

When a user runs our application (`zf`) with these parameters, we'd like to call action
`resetpassword` of `Application\Controller\IndexController`.

> ## Note
We will use `zf` to depict the entry point for your application, it can be shell script in
application bin folder or simply an alias for `php public/index.php`

First we need to create a **route definition**:

``` sourceCode
user resetpassword <userEmail>
```

This simple route definition expects exactly 3 arguments: a literal "user", literal "resetpassword"
followed by a parameter we're calling "userEmail". Let's assume we also accept one optional
parameter, that will turn on verbose operation:

``` sourceCode
user resetpassword [--verbose|-v] <userEmail>
```

Now our console route expects the same 3 parameters but will also recognise an optional `--verbose`
flag, or its shorthand version: `-v`.

> ## Note
The order of flags is ignored by `Zend\Console`. Flags can appear before positional parameters,
after them or anywhere in between. The order of multiple flags is also irrelevant. This applies both
to route definitions and the order that flags are used on the command line.

Let's use the definition above and configure our console route. Console routes are automatically
loaded from the following location inside config file:

``` sourceCode
array(
    'router' => array(
        'routes' => array(
            // HTTP routes are defined here
        )
    ),

    'console' => array(
        'router' => array(
            'routes' => array(
                // Console routes go here
            )
        )
    ),
)
```

Let's create our console route and point it to
`Application\Controller\IndexController::resetpasswordAction()`

``` sourceCode
// we could define routes for Application\Controller\IndexController in Application module config
file
// which is usually located at modules/application/config/module.config.php
array(
    'console' => array(
        'router' => array(
            'routes' => array(
                'user-reset-password' => array(
                    'options' => array(
                        'route'    => 'user resetpassword [--verbose|-v] <userEmail>',
                        'defaults' => array(
                            'controller' => 'Application\Controller\Index',
                            'action'     => 'resetpassword'
                        )
                    )
                )
            )
        )
    )
)
```

## Handling console requests

When a user runs our application from command line and arguments match our console route, a
`controller` class will be instantiated and an `action` method will be called, just like it is with
http requests.

We will now add `resetpassword` action to `Application\Controller\IndexController`:

``` sourceCode
<?php
namespace Application\Controller;

use Zend\Mvc\Controller\AbstractActionController;
use Zend\View\Model\ViewModel;
use Zend\Console\Request as ConsoleRequest;
use Zend\Math\Rand;

class IndexController extends AbstractActionController
{
    public function indexAction()
    {
        return new ViewModel(); // display standard index page
    }

    public function resetpasswordAction()
    {
        $request = $this->getRequest();

        // Make sure that we are running in a console and the user has not tricked our
        // application into running this action from a public web server.
        if (!$request instanceof ConsoleRequest){
            throw new \RuntimeException('You can only use this action from a console!');
        }

        // Get user email from console and check if the user used --verbose or -v flag
        $userEmail   = $request->getParam('userEmail');
        $verbose     = $request->getParam('verbose') || $request->getParam('v');

        // reset new password
        $newPassword = Rand::getString(16);

        //  Fetch the user and change his password, then email him ...
        // [...]

        if (!$verbose) {
            return "Done! $userEmail has received an email with his new password.\n";
        }else{
            return "Done! New password for user $userEmail is '$newPassword'. It has also been
emailed to him. \n";
        }
    }
}
```

We have created `resetpasswordAction()` than retrieves current request and checks if it's really
coming from the console (as a precaution). In this example we do not want our action to be invocable
from a web page. Because we have not defined any http route pointing to it, it should never be
possible. However in the future, we might define a wildcard route or a 3rd party module might
erroneously route some requests to our action - that is why we want to make sure that the request is
always coming from a Console environment.

All console arguments supplied by the user are accessible via `$request->getParam()` method. Flags
will be represented by a booleans, where `true` means a flag has been used and `false` otherwise.

When our action has finished working it returns a simple `string` that will be shown to the user in
console window.

## Adding console usage info

It is a common practice for console application to display usage information when run for the first
time (without any arguments). This is also handled by `Zend\Console` together with `MVC`.

Usage info in ZF2 console applications is provided by loaded modules
&lt;zend.module-manager.intro&gt;. In case no console route matches console arguments,
`Zend\Console` will query all loaded modules and ask for their console usage info.

Let's modify our `Application\Module` to provide usage info:

``` sourceCode
<?php

namespace Application;

use Zend\ModuleManager\Feature\AutoloaderProviderInterface;
use Zend\ModuleManager\Feature\ConfigProviderInterface;
use Zend\ModuleManager\Feature\ConsoleUsageProviderInterface;
use Zend\Console\Adapter\AdapterInterface as Console;

class Module implements
    AutoloaderProviderInterface,
    ConfigProviderInterface,
    ConsoleUsageProviderInterface   // <- our module implement this feature and provides console
usage info
{
    public function getConfig()
    {
        // [...]
    }

    public function getAutoloaderConfig()
    {
        // [...]
    }

    public function getConsoleUsage(Console $console)
    {
        return array(
            // Describe available commands
            'user resetpassword [--verbose|-v] EMAIL'    => 'Reset password for a user',

            // Describe expected parameters
            array( 'EMAIL',            'Email of the user for a password reset' ),
            array( '--verbose|-v',     '(optional) turn on verbose mode'        ),
        );
    }
}
```

Each module that implements `ConsoleUsageProviderInterface` will be queried for console usage info.
On route mismatch, all info from all modules will be concatenated, formatted to console width and
shown to the user.

> ## Note
The order of usage info displayed in the console is the order modules load. If you want your
application to display important usage info first, change the order your modules are loaded.
