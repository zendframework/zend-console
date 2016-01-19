# Console routes and routing

Zend Framework 2 has native MVC integration with console&lt;zend.console.introduction&gt;, which
means that command line arguments are read and used to determine the appropriate action controller
&lt;zend.mvc.controllers&gt; and action method that will handle the request. Actions can perform any
number of task prior to returning a result, that will be displayed to the user in his console
window.

There are several routes you can use with Console. All of them are defined in
`Zend\Mvc\Router\Console\*` classes.

## Router configuration

All Console Routes are automatically read from the following configuration location:

```php
// This can sit inside of modules/Application/config/module.config.php or any other module's config.
array(
    'router' => array(
        'routes' => array(
            // HTTP routes are here
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

Console Routes will only be processed when the application is run inside console (terminal) window.
They have no effect in web (http) request and will be ignored. It is possible to define only HTTP
routes (only web application) or only Console routes (which means we want a console-only application
which will refuse to run in a browser).

A single route can be described with the following array:

```php
// inside config.console.router.routes:
// [...]
'my-first-route' => array(
    'type'    => 'simple',       // <- simple route is created by default, we can skip that
    'options' => array(
        'route'    => 'foo bar',
        'defaults' => array(
            'controller' => 'Application\Controller\Index',
            'action'     => 'password'
        )
    )
)
```

We have created a `simple` console route with a name `my-first-route`. It expects two parameters:
`foo` and `bar`. If user puts these in a console,
`Application\Controller\IndexController::passwordAction()` action will be invoked.

## Basic route

This is the default route type for console. It recognizes the following types of parameters:

- \[Literal parameters\](literal-params) (i.e. `create object (external|internal)`)
- \[Literal flags\](literal-flags) (i.e. `--verbose --direct [-d] [-a]`)
- \[Positional value parameters\](value-positional) (i.e. `create <modelName> [<destination>]`)
- \[Value flags\](value-flags) (i.e. `--name=NAME [--method=METHOD]`)

### Literal parameters

These parameters are expected to appear on the command line exactly the way they are spelled in the
route. For example:

```php
'show-users' => array(
    'options' => array(
        'route'    => 'show users',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'show'
        )
    )
)
```

This route will **only** match for the following command line

```php
> zf show users
```

It expects **mandatory literal parameters** `show users`. It will not match if there are any more
params, or if one of the words is missing. The order of words is also enforced.

We can also provide **optional literal parameters**, for example:

```php
'show-users' => array(
    'options' => array(
        'route'    => 'show [all] users',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'show'
        )
    )
)
```

Now this route will match for both of these commands:

```php
> zf show users
zf show all users
```

We can also provide **parameter alternative**:

```php
'show-users' => array(
    'options' => array(
        'route'    => 'show [all|deleted|locked|admin] users',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'show'
        )
    )
)
```

This route will match both without and with second parameter being one of the words, which enables
us to capture commands such:

```php
> zf show users
zf show locked users
zf show admin users
etc.
```

> ## Note
Whitespaces in route definition are ignored. If you separate your parameters with more spaces, or
separate alternatives and pipe characters with spaces, it won't matter for the parser. The above
route definition is equivalent to: `show [  all | deleted | locked | admin  ]   users`

### Literal flags

Flags are a common concept for console tools. You can define any number of optional and mandatory
flags. The order of flags is ignored. The can be defined in any order and the user can provide them
in any other order.

Let's create a route with **optional long flags**

```php
'check-users' => array(
    'options' => array(
        'route'    => 'check users [--verbose] [--fast] [--thorough]',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'check'
        )
    )
)
```

This route will match for commands like:

```php
> zf check users
zf check users --fast
zf check users --verbose --thorough
zf check users --thorough --fast
```

We can also define one or more **mandatory long flags** and group them as an alternative:

```php
'check-users' => array(
    'options' => array(
        'route'    => 'check users (--suspicious|--expired) [--verbose] [--fast] [--thorough]',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'check'
        )
    )
)
```

This route will **only match** if we provide either `--suspicious` or `--expired` flag, i.e.:

```php
> zf check users --expired
zf check users --expired --fast
zf check users --verbose --thorough --suspicious
```

We can also use **short flags** in our routes and group them with long flags for convenience, for
example:

```php
'check-users' => array(
    'options' => array(
        'route'    => 'check users [--verbose|-v] [--fast|-f] [--thorough|-t]',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'check'
        )
    )
)
```

Now we can use short versions of our flags:

```php
> zf check users -f
zf check users -v --thorough
zf check users -t -f -v
```

### Positional value parameters

Value parameters capture any text-based input and come in two forms - positional and flags.

**Positional value parameters** are expected to appear in an exact position on the command line.
Let's take a look at  
the following route definition:

```php
'delete-user' => array(
    'options' => array(
        'route'    => 'delete user <userEmail>',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'delete'
        )
    )
)
```

This route will match for commands like:

```php
> zf delete user john@acme.org
zf delete user betty@acme.org
```

We can access the email value by calling `$this->getRequest()->getParam('userEmail')` inside of our
controller action (you can \[read more about accessing values here\](reading-values))

We can also define **optional positional value parameters** by adding square brackets:

```php
'delete-user' => array(
    'options' => array(
        'route'    => 'delete user [<userEmail>]',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'delete'
        )
    )
)
```

In this case, `userEmail` parameter will not be required for the route to match. If it is not
provided, `userEmail` parameter will not be set.

We can define any number of positional value parameters, for example:

```php
'create-user' => array(
    'options' => array(
        'route'    => 'create user <firstName> <lastName> <email> <position>',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'create'
        )
    )
)
```

This allows us to capture commands such as:

```php
> zf create user Johnny Bravo john@acme.org Entertainer
```

> ## Note
Command line arguments on all systems must be properly escaped, otherwise they will not be passed to
our application correctly. For example, to create a user with two names and a complex position
description, we could write something like this:
```php
zf create user "Johnan Tom" Bravo john@acme.org "Head of the Entertainment Department"
```

### Value flag parameters

Positional value parameters are only matched if they appear in the exact order as described in the
route. If we do not want to enforce the order of parameters, we can define **value flags**.

**Value flags** can be defined and matched in any order. They can digest text-based values, for
example:

```php
'find-user' => array(
    'options' => array(
        'route'    => 'find user [--id=] [--firstName=] [--lastName=] [--email=] [--position=] ',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'find'
        )
    )
)
```

This route will match for any of the following routes:

```php
> zf find user
zf find user --id 29110
zf find user --id=29110
zf find user --firstName=Johny --lastName=Bravo
zf find user --lastName Bravo --firstName Johny
zf find user --position=Executive --firstName=Bob
zf find user --position "Head of the Entertainment Department"
```

> ## Note
The order of flags is irrelevant for the parser.

> ## Note
The parser understands values that are provided after equal symbol (=) and separated by a space.
Values without whitespaces can be provided after = symbol or after a space. Values with one more
whitespaces however, must be properly quoted and written after a space.

In previous example, all value flags are optional. It is also possible to define **mandatory value
flags**:

```php
'rename-user' => array(
    'options' => array(
        'route'    => 'rename user --id= [--firstName=] [--lastName=]',
        'defaults' => array(
            'controller' => 'Application\Controller\Users',
            'action'     => 'rename'
        )
    )
)
```

The `--id` parameter **is required** for this route to match. The following commands will work with
this route:

```php
> zf rename user --id 123
zf rename user --id 123 --firstName Jonathan
zf rename user --id=123 --lastName=Bravo
```

## Catchall route

This special route will catch all console requests, regardless of the parameters provided.

```php
'default-route' => array(
    'type'     => 'catchall',
    'options' => array(
        'route'    => '',
        'defaults' => array(
            'controller' => 'Application\Controller\Index',
            'action'     => 'consoledefault'
        )
    )
)
```

> ## Note
This route type is rarely used. You could use it as a last console route, to display usage
information. Before you do so, read about the preferred way of displaying console usage information
&lt;zend.console.modules&gt;. It is the recommended way and will guarantee proper inter-operation
with other modules in your application.

## Console routes cheat-sheet

<table>
<colgroup>
<col width="25%" />
<col width="20%" />
<col width="53%" />
</colgroup>
<thead>
<tr class="header">
<th align="left">Param type</th>
<th align="left">Example route definition</th>
<th align="left">Explanation</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left"><strong>Literal params</strong></td>
<td align="left"></td>
<td align="left"></td>
</tr>
<tr class="even">
<td align="left">Literal</td>
<td align="left"><code>foo bar</code></td>
<td align="left">&quot;foo&quot; followed by &quot;bar&quot;</td>
</tr>
<tr class="odd">
<td align="left">Literal alternative</td>
<td align="left"><code>foo (bar|baz)</code></td>
<td align="left">&quot;foo&quot; followed by &quot;bar&quot; or &quot;baz&quot;</td>
</tr>
<tr class="even">
<td align="left">Literal, optional</td>
<td align="left"><code>foo [bar]</code></td>
<td align="left">&quot;foo&quot;, optional &quot;bar&quot;</td>
</tr>
<tr class="odd">
<td align="left">Literal, optional alternative</td>
<td align="left"><code>foo [bar|baz]</code></td>
<td align="left">&quot;foo&quot;, optional &quot;bar&quot; or &quot;baz&quot;</td>
</tr>
<tr class="even">
<td align="left"><strong>Flags</strong></td>
<td align="left"></td>
<td align="left"></td>
</tr>
<tr class="odd">
<td align="left">Flag long</td>
<td align="left"><code>foo --bar</code></td>
<td align="left">&quot;foo&quot; as first parameter, &quot;--bar&quot; flag before or after</td>
</tr>
<tr class="even">
<td align="left">Flag long, optional</td>
<td align="left"><code>foo [--bar]</code></td>
<td align="left">&quot;foo&quot; as first parameter, optional &quot;--bar&quot; flag before or
after</td>
</tr>
<tr class="odd">
<td align="left">Flag long, optional, alternative</td>
<td align="left"><code>foo [--bar|--baz]</code></td>
<td align="left">&quot;foo&quot; as first parameter, optional &quot;--bar&quot; or
&quot;--baz&quot;, before or after</td>
</tr>
<tr class="even">
<td align="left">Flag short</td>
<td align="left"><code>foo -b</code></td>
<td align="left">&quot;foo&quot; as first parameter, &quot;-b&quot; flag before or after</td>
</tr>
<tr class="odd">
<td align="left">Flag short, optional</td>
<td align="left"><code>foo [-b]</code></td>
<td align="left">&quot;foo&quot; as first parameter, optional &quot;-b&quot; flag before or
after</td>
</tr>
<tr class="even">
<td align="left">Flag short, optional, alternative</td>
<td align="left"><code>foo [-b|-z]</code></td>
<td align="left">&quot;foo&quot; as first parameter, optional &quot;-b&quot; or &quot;-z&quot;,
before or after</td>
</tr>
<tr class="odd">
<td align="left">Flag long/short alternative</td>
<td align="left"><code>foo [--bar|-b]</code></td>
<td align="left">&quot;foo&quot; as first parameter, optional &quot;--bar&quot; or &quot;-b&quot;
before or after</td>
</tr>
<tr class="even">
<td align="left"><strong>Value parameters</strong></td>
<td align="left"></td>
<td align="left"></td>
</tr>
<tr class="odd">
<td align="left">Value positional param</td>
<td align="left"><code>foo &lt;bar&gt;</code></td>
<td align="left">&quot;foo&quot; followed by any text (stored as &quot;bar&quot; param)</td>
</tr>
<tr class="even">
<td align="left">Value positional param, optional</td>
<td align="left"><code>foo [&lt;bar&gt;]</code></td>
<td align="left">&quot;foo&quot;, optionally followed by any text (stored as &quot;bar&quot;
param)</td>
</tr>
<tr class="odd">
<td align="left">Value Flag</td>
<td align="left"><code>foo --bar=</code></td>
<td align="left">&quot;foo&quot; as first parameter, &quot;--bar&quot; with a value, before or
after</td>
</tr>
<tr class="even">
<td align="left">Value Flag, optional</td>
<td align="left"><code>foo [--bar=]</code></td>
<td align="left">&quot;foo&quot; as first parameter, optionally &quot;--bar&quot; with a value,
before or after</td>
</tr>
<tr class="odd">
<td align="left"><strong>Parameter groups</strong></td>
<td align="left"></td>
<td align="left"></td>
</tr>
<tr class="even">
<td align="left">Literal params group</td>
<td align="left"><code>foo (bar|baz):myParam</code></td>
<td align="left">&quot;foo&quot; followed by &quot;bar&quot; or &quot;baz&quot; (stored as
&quot;myParam&quot; param)</td>
</tr>
<tr class="odd">
<td align="left">Literal optional params group</td>
<td align="left"><code>foo [bar|baz]:myParam</code></td>
<td align="left">&quot;foo&quot; followed by optional &quot;bar&quot; or &quot;baz&quot; (stored as
&quot;myParam&quot; param)</td>
</tr>
<tr class="even">
<td align="left">Long flags group</td>
<td align="left"><code>foo (--bar|--baz):myParam</code></td>
<td align="left">&quot;foo&quot;, &quot;bar&quot; or &quot;baz&quot; flag before or after (stored as
&quot;myParam&quot; param)</td>
</tr>
<tr class="odd">
<td align="left">Long optional flags group</td>
<td align="left"><code>foo [--bar|--baz]:myParam</code></td>
<td align="left">&quot;foo&quot;, optional &quot;bar&quot; or &quot;baz&quot; flag before or after
(as &quot;myParam&quot; param)</td>
</tr>
<tr class="even">
<td align="left">Short flags group</td>
<td align="left"><code>foo (-b|-z):myParam</code></td>
<td align="left">&quot;foo&quot;, &quot;-b&quot; or &quot;-z&quot; flag before or after (stored as
&quot;myParam&quot; param)</td>
</tr>
<tr class="odd">
<td align="left">Short optional flags group</td>
<td align="left"><code>foo [-b|-z]:myParam</code></td>
<td align="left">&quot;foo&quot;, optional &quot;-b&quot; or &quot;-z&quot; flag before or after
(stored as &quot;myParam&quot; param)</td>
</tr>
</tbody>
</table>


