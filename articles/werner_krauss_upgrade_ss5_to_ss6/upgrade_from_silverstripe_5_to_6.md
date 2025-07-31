# Updating to Silverstripe 6
In June 2025 Silverstripe 6 was released. A new major version, cool. Some cool new features and many small changes under the hood for an up-to-date codebase. 

The most significant changes are that it bumped the minimum Version of PHP to 8.3 and many classes got renamed. So updating could be cumbersome? Not really. As many people face the problems of upgrading code, some clever folks invented Rector, a tool for upgrading PHP with defined and tested rules, so-called rectors. This can help us to e.g. rename classes automatically or to convert code to newer constructs, e.g. `match `instead of `switch`, `string_starts_with` instead of checking `str_pos`, etc...

This document will lead you through the progress of upgrading a site from Silverstripe 5 to Silverstripe 6 and will explain the steps and tools I recommend for doing this task.

## Step 1: Update Code And All Modules To Latest SS5

Before upgrading you should update your project, including all modules to the latest version of Silverstripe CMS 5. Your IDE will complain about a lot of deprecations, most of them can be handled automatically.

## Step 2: Install Rector

For upgrading PHP code, I use a tool called [Rector](https://getrector.com/). It uses PHP's abstract syntax tree (AST) to analyse your code and can change code much more precisely than any search-and-replace can ever do, cause it knows if it's changing a class name or a variable name, etc. Therefor you can write pretty specific rules, rectors, to change code in certain circumstances.

I started to collect some [rectors specifically for Silverstripe CMS](https://packagist.org/packages/wernerkrauss/silverstripe-rector). Feel free to suggest enhancements or to PR your own rectors.

We'll use this tool to upgrade our code.

### install Rector

Rector should be installed as a developer dependency:

```bash
composer require wernerkrauss/silverstripe-rector:^1 --dev
```

It will automatically install Rector, PHPStan and other required packages.

Next, you'll need to add a _rector.php_ to your project root. Here is my boilerplate:

```php
<?php

declare(strict_types=1);

use Netwerkstatt\SilverstripeRector\Set\SilverstripeLevelSetList;
use Netwerkstatt\SilverstripeRector\Set\SilverstripeSetList;
use Rector\Config\RectorConfig;

return RectorConfig::configure()
    ->withPaths([
        __DIR__ . '/app/_config.php',
        __DIR__ . '/app/src',
    ])
    // uncomment to reach your current PHP version
    ->withPhpSets()
    ->withSets([
        SilverstripeSetList::CODE_STYLE,
        SilverstripeLevelSetList::UP_TO_SS_6_0
    ])
    ->withTypeCoverageLevel(0)
    ->withDeadCodeLevel(0)
    ->withCodeQualityLevel(0);


```

in the `withPaths()` method we can configure all the paths of our code, Rector should check. `withPHPSets()` automatically updates your code to the currently used PHP version. In `withSets()` we can define some sets of rectors, in this case for Silverstripe code style (e.g. use `::create()` where possible) and for upgrading to the latest and greatest version of Silverstripe.

The remaining directives help you to improve your code step by step with better type coverage, code quality or less unneeded code.

[For a deeper dive into configuring Rector, you can check its documentation](https://getrector.com/documentation).

Running Rector on an old code base will change a lot. Too much for checking if everything is still right. So it's recommended that you change one thing at a time, commit the changes to git and then change the next thing. So first, you can comment out the Silverstripe sets and only update your code to PHP8.3. 

## Step 3: Update Code In The Current Project As Much As Possible

Now let's run Rector the first time. Make sure that your last code changes are checked into git, so you can roll back at any time. You can now run

```bash
vendor/bin/rector --dry-run
```

in your CLI.

With `--dry-run` we tell Rector to show us, what it would like to change. If we're happy with the changes, we can remove that parameter and let Rector change our code.

Then you can commit the changes, enable another Rector or setlist and repeat.

As we're still on SS5, you might use the setlist `SilverstripeLevelSetList::UP_TO_SS_5_4`.

## Step 4: Check And Update Dependencies In SS5

Now that our project's code is improved and hopefully all deprecations are fixed (we'll rename classes in Step 4.2), we need to update composer's requirements when switching to Silverstripe's new major version.

### Step 4.1: Update Modules
Most Silverstripe projects use a lot of modules, some of them might be already updated, others not. So the next step is to check the required modules in your _composer.json_ for updates. I do this manually on packagist.


If a module doesn't have a SS6 compatible release, I also go to its GitHub page and check on "Insights → Network" if there are any recent forks that already updated that module. Sometimes this saves duplicate work.

Modules that are already compatible with SS6 can be updated, others need to be removed for now.

### Step 4.2: Update Your Code With Rector To SS6, Check For Errors
So now that you installed SS6 to your project, running dev/build will throw a bunch of errors. That's the time to upgrade your code. Enable `SilverstripeLevelSetList::UP_TO_SS_6_0` in your _rector.php_ and run Rector.

 Once dev/build works, we're one step further, but still not done. Make sure your code works as expected and use tools for linting or static analysis to find errors in your code and raise its quality.

Note: build tasks have massive changes, and I don't have (yet) Rector rules for them. As Tasks are now full Symfony CLI scripts, you need to spend some time to think about input parameters etc.

## Step 5: Update Missing Modules Yourself

So there are some modules still not available for SS6? You can either complain or remember what open source is all about: collaboration. So feel free to update those modules. It's not really harder than upgrading your own code.

### Step 5.1: Fork The Project
For changing code in a third-party module, you need to fork it on GitHub.

### Step 5.2: Clone The Module
I tend to clone the module in my project root so that I can reach the code more comfortably.

### Step 5.3: Create A Branch For Updating
In the cloned module, create a branch, e.g. 'silverstripe6', for upgrading the module.

### Step 5.4: Add It To Your Project
You can add a module (with all its dependencies) to your project with Composer's path directive. This is cool for upgrading. For delivering the project, I prefer to install a branch from my repository's VCS directly until the upgrade is released.

```composer.json
    "repositories": [
        {
            "type": "path",
            "url": "./module-to-update"
        },
```

### Step 5.5: Update The Module's Code Using Rector
Add the module's path to your _rector.php_ and update its code step by step. Make sure to commit after each iteration.

Once you're happy with your results, push the code to your fork on GitHub and let's go on to the next step.

### Step 5.6: Require The Updated Module As A Fork

Let's change the repository in _composer.json_ to

```composer.json
    "repositories": [
        {
            "type": "git",
            "url": "https://github.com/myhandle/module.git"
        },
```

and change the requirement to

```composer.json
    "vendor/module": "dev-silverstripe6 as 6.0",
```

Here, `dev-silverstripe6` needs to match the name of your branch, in this example `silverstripe6`. The `as x.y.z` tells Composer to treat your fork as version x.y.z of the module. If the current version of the module is 5.4, and I have breaking changes (e.g. classes got renamed), I choose to treat my fork as a new major version.

By the way:
 
 What do you get when you throw a piano onto an army base? A flat major.

Sorry. Not.

### Step 5.7: Share Your Work
Now that the module works fine, it's essential that you create a pull request on the original module, so others can also upgrade their projects to Silverstripe 6.

## What's Next?
* Test your code<br>Of course, you already have unit tests that can guarantee everything works as expected.
* Add more code quality tools like linting or static analysis. <br>This annoys a lot in the beginning, but once you're through the first frustrations, those tools help you find buggy code more easily. It's really worth the work!
* If you find some recurring changes while updating, feel free to raise an issue or create your own Rector and submit a PR to _silverstripe-rector_.

## Conclusion

The new major version 6 of our good ol' friend Silverstripe CMS comes with a lot of changes, but there are tools that help us to update our code as much as possible. With Rector, we can automate many parts of the upgrade task and get better code quality for free.