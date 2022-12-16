# Using Silverstripe CMS with the PHP built-in-server
## or "Why a virtualization is not always the best answer for your local development setup"

Using similar server setups on your local development machine and on production servers became popular over the last years. DevOps as best practice defining a transparent and reconstructable process for continuous integration and delivery brought virtualization to the developers' daily business. Modern containerization decreased the costs of time and computer resources to run complex setups of services interacting with each other. But what if your project setup does not require many services to work with? What if in most cases the simple PHP and database stack is sufficient? The answer might be already integrated in PHP.

KISS [1] and similar principles [2] are always a nice goal to keep in mind. Especially on green-field-projects, where the growth of a project will increase the complexity sooner or later.

### Clean and simple Silverstripe setup

The requirements for Silverstripe are straightforward:
    
    * a modern PHP version
    * a MySQL database
    * a webserver like apache or nginx
    
Wait, do we need a webserver in development?

No, because PHP as a built-in-server [3]. To test it, just create a php file, for example this `info.php`:

```php
<?php
phpinfo();
```

Then run on the command-line:

```sh
$ php -S localhost:3000 info.php
[Sat Dec 24 14:09:16 2022] PHP 8.1.12 Development Server (http://localhost:3000) started
```

The php script is now served on `http://localhost:3000/`.

### Does it work with Silverstripe CMS?

Yes, but you have to install a module [4] [5] to start the server:

```sh
$ cd my-project
$ composer require pstaender/silverstripe-serve
```

Now you can start the Silverstripe project with:

```sh
$ vendor/bin/serve
```

You can also change host and port if you like:

```sh
$ vendor/bin/serve --host 127.0.0.1 --port 8000
```

## Nice to have: A server composer command

It's convenient to have all your development commands in one place: the composer is a good place for that. To start the server with `composer server` instead of `vendor/bin/serve`, add to your `composer.json`:

```json
{
    "scripts": {
        "server": [
            "Composer\\Config::disableProcessTimeout",
            "@php serve"
        ]
    },
}
```

## XDebug

You can also use XDebug with the built-in-server. Just add those lines to your `php.ini` (check with `phpinfo()` the correct path of php.ini for the built-in-server):

```
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.discover_client_host=1
```

### Is the Built-in-server more suitable for local development?

It depends on your project and your general development process. Using the build-in-server enables faster development, because no extra webserver setup is required. The speed of php processing is excellent because it runs natively on your machine and the minimal setup reduces the complexity of your project. Because of one less service and php running natively on your machine  less hardware resources are required - and that means more battery life. The server runs on Linux, Mac and Windows.

### Easy to use with different PHP version

To use different PHP version on the same environment, you can set up your own php version switch. In this example we have PHP 8.1 and PHP 7.4 installed.

On your Mac (php is installed with homebrew) define these bash function in your `~/.bash_profile` or `~/.zshrc`:

```bash
use_php7.4 () {
  brew unlink php@8.1
  brew link php@7.4 --force --overwrite
  export PATH="/opt/homebrew/opt/php@7.4/bin:$PATH"
  export PATH="/opt/homebrew/opt/php@7.4/sbin:$PATH"
}

use_php8.1 () {
  brew unlink php@7.4
  brew link php@8.1 --force --overwrite
  export PATH="/opt/homebrew/opt/php@8.1/bin:$PATH"
  export PATH="/opt/homebrew/opt/php@8.1/sbin:$PATH"
}
```

After reloading your shell, you can now switch between the versions:

```sh
$ use_php7.4
$ use_php8.1
```

On Debian/Ubuntu you can change the default PHP version with:

```sh
$ sudo update-alternatives --set php /usr/bin/php7.4
$ sudo update-alternatives --set php /usr/bin/php8.1
```

### Opinion: Maybe you don't need docker, vagrant etc.

Emulation and containerization are great tools and useful for more complex projects dealing with several services. But let's remember that Silverstripe has its origin in classic LAMP web hosting. That means by default the only service which is required by Silverstripe is MySQL.

The importance of speed in relation with actual page visits in production is often overrated (are we dealing with 100 or 100.000 requests per minute?), but underrated in local development. Why should I use a slower and more resource-hungry setup [6] when I need to debug code, do testing without caching and having a different configuration in development than in production anyway? My personal trade-off in development is: Better a small and simple than a large and complex setup. Using the built-in-server is one step closer to this goal.

### References

[1]: https://en.wikipedia.org/wiki/KISS_principle
[2]: http://www.radicalsimpli.city/
[3]: https://www.php.net/manual/en/features.commandline.webserver.php
[4]: [Silverstripe serve module for Silverstripe v4](https://packagist.org/packages/pstaender/silverstripe-serve#dev-master)
[5]: [The official module (currently not for Silverstripe v4)](https://github.com/silverstripe/silverstripe-serve/)
[6]: https://accesto.com/blog/docker-on-mac-how-to-speed-it-up/
[7]: [Lightning talk by Philipp Ständer](https://www.youtube.com/watch?v=PKiw0geTLss)