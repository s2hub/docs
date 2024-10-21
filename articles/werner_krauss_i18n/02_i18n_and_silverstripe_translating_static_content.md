# How to create multilingual sites in Silverstripe CMS
Part 2 - i18n and Silverstripe CMS

In the first part of this series we defined what i18n is and what parts of a software project are affected by i18n. Today we will have a look at how Silverstripe CMS can help you with i18n and how you can translate static content in your webapp.

## i18n and Silverstripe CMS

How can Silverstripe help you with i18n and which tools can you use for making webapps for the international market?

The core class for internationalisation is called - you guessed it -  `i18n`. That's where all logic for static translation in code and templates live.  The key methods are `_t()` in PHP and `<%t` for templates.

It's called _static translation_, because the translation is stored in code (your version control system) and config variables and cannot be changed by editors. 
That means, static translation is mainly affecting the user interface (e.g. texts like: 1 product added to the cart, contact me now, you are here...)

If you want to translate CMS content stored in your database, then a module called Fluent is your friend. Here you can define multiple locales and change the content for each locale in the CMS. That's called dynamic translation and will be part of the next article in this series. First things first...

Of course Silverstripe CMS has some good [documentation about i18n in the developer guides](https://docs.silverstripe.org/en/5/developer_guides/i18n/).

## The basics: class i18n

Let's have a deeper look at the basic class `i18n`. It stores different data and keeps that data available for you and your code. First it stores the current locale you can set either globally or based on the URL of the current page. Then it contains the static `_t()` method for translating static properties in your PHP code and templates.

So, let's set the default locale of your webapp. You can do this in your config.yml:

```yaml
SilverStripe\i18n\i18n:
  default_locale: 'de_DE'
```

or in your PHP code:

```php
use SilverStripe\i18n\i18n;
i18n::set_locale('de_DE');
```


What if you want to translate your own model classes? Then you can use the interface `i18nEntitiyProvider` for providing an array with all entities you want to translate in a standardised way: the singlular and plural names and rules for pluralisation (e.g. one tree, two trees, or in German "ein Baum, zwei Bäume").

That interface is mainly used by `DataObject` and can be overwritten in your subclasses. Most of the time all entities provided by `DataObject` are enough for your needs. But if you have to translate e.g. configuration arrays, this is a nice way to collect all items that need to be translated programmatically. The source code has a nice example for that:

 ```php
class MyTestClass implements i18nEntityProvider
{
  public function provideI18nEntities()
  {
    $entities = [];
    foreach($this->config()->get('my_static_array') as $key => $value) {
      $entities["MyTestClass.my_static_array_{$key}"] = $value;
    }
    $entities["MyTestClass.PLURALS"] = [
      'one' => 'A test class',
      'other' => '{count} test classes',
    ]
    return $entities;
  }
}
```

So we have some entities we want to translate, but how is it actually done? Well, let's move on to the next section.

## The process of translating static content

The `_t` method is used for translating static content, in both, PHP code and templates. You need to provide a translation key and a fallback translation, if your current locale doesn't have that key translated.

```php
_t(__CLASS__ . '.PageTitle', 'Item Title')
```

or 

```php 
return $this->error(_t(__CLASS__ . '.ItemNotFound', 'Item not found.'));
```

Note that Silverstripe defined a helper function for `SilverStripe\i18n\i18n::_t()`. You only need to use `_t()` in your code.

`_t` can also replace placeholders in your translations, cause depending on the locale you want to add the number of items at the start, the beginning or the end of your translated text. The syntax for that is: 

```php
_t('MyObject.SomeKey', 'Added {count} items', [ 'count' => $count ]);
```

You see, that the third parameter for `_t` is for passing the parameters to replace the placeholders with the strings provided. That's also possible in templates, though it doesn't look that common.

One word on plurals... With the example above, you could also write a plural translation for the same key. The syntax for that is:

```php
// Plurals are invoked via a `|` pipe-delimeter with a {count} argument
_t('MyObject.SomeKey', 'Added an item|Added {count} items', [ 'count' => $count ]);
```
You see, that `{count}` is only used for the plural version of the translation. 

I think it's a good practice to give English keys and fallback translations to your static content, even if you're working on a customer project. IMHO it's a good standard, that English is the source language in your code. You never know who will work at that project later and if they speak the same language as you do.

## Where are all those translations stored?

Silverstripe CMS stores the translated values in yml-files follwing [Ruby's i18n convention](https://guides.rubyonrails.org/i18n.html#pluralization) in the _lang/_ folder of each module or in _app/lang_. This way they are part of your code, can be edited in your IDE and stored in your version control system. On `?flush`, Silverstripe checks the yml files and caches all translation in a faster cache. This means, you have to tell Silverstripe by flushing the cache if you've made changes in your yml translations.

The translation for e.g. German is stored in _lang/de.yml_, if you need to overwrite some strings for Austrian de_AT, you can do this in _lang/de_AT.yml_. If a string isn't available in de_AT, the framework automatically falls back to de.yml. That's pretty handy, isn't it?

## Some sugar: default translations for DataObjects

Silverstripe CMS is built for i18n and tries to help you as much as possible to make your work as easy as possible. You don't need to define e.g. translations in PHP for every field label from `$db`, `$has_one` etc, there is some syntactic sugar for you to use in your yml files:

SINGULARNAME, PLURALNAME and PLURALS for naming the DataObject. You can define this either in PHP or yml:

```yaml
    MyDataObject:
      SINGULARNAME: 'My Data Object'
      PLURALNAME: 'My Data Objects'
      PLURALS:
        one: 'A data object'
        other: '{count} data objects'
```

If you use PHP, PLURALS are automatically generated by DataObject's provideI18nEntities() method when collecting the entities (see below) so you don't need to define it yourself.

```php
    private static $singular_name = 'Category';

    private static $plural_name = 'Categories';
```




Field labels and relation lables can be defined like `db_Title`, `has_one_Album` (when your relation is named Album), etc...
There is no need to clutter the PHP  code with defining translations for field labels anymore. 

```yml
  SilverShop\Model\Address:
    PLURALNAME: Addresses
    SINGULARNAME: Address
    db_Address: Address
    db_AddressLine2: 'Address Line 2 (optional)'
    db_City: City
    db_Company: Company
    db_Country: Country
    db_FirstName: 'First Name'
    db_Phone: 'Phone Number'
    db_PostalCode: 'Postal Code'
    db_State: State
    db_Surname: Surname
    has_one_Member: Member
 ```

This works for all db fields and relations, except `$belongs_to` and `$manymany_trough`.

Note: for every key you could also define plural forms like:
```yml
Address:
  Company:
    one: Company
    other: Companies
```

## Collecting all the translations from code and templates

When you have all the static content in your code and Silverstripe templates translated, you can copy them over to your yml files. That's kind of a boring work to do, and therefore Silverstripe CMS helps you with this by providing the `I18nTextCollectorTask`. It screens all your PHP and .ss files for things to translate and writes it into the defined yml file.

You can run it by calling the task via URL or CLI, e.g. `https://myproject.local/dev/tasks/i18nTextCollectorTask`

If you want to run it for one or more modules, you can run it with the module parameter, e.g. `https://myproject.local/dev/tasks/i18nTextCollectorTask?module=app,vendor%2Fmodule`

And if you want to collect from one or more themes, you can run it like `https://myproject.local/dev/tasks/i18nTextCollectorTask?module=themes:mytheme,app,vendor%2Fmodule`

[//]: # (TODO: Still correct? A requirement for I18nTextCollectorTask is PHPUnit, which should be installed in your development environment.) 

Ok, now we have all static text translated in PHP and the templates, now we need to grab them all so we have them in yml and can send them to our translator.

## How to fix missing translations in 3rd-party modules?

Now that we know, that all the translations of static content are just strings in yml files, we know how to fix translations in 3rd-party modules in our project without forking the module.

It's really easy, because language yml files in your app module overrule all module's language strings - similar to configurations in yml. Knowing that is the key to fix missing or incorrect translations in modules:

1. find the missing string in templates or code
2. add or modify your language's yml file in _app/lang_
3. flush caches, so Silverstripe CMS notices the new translation

Of course, if you add translations or fix errors, please make a pull request and give your changes back to the community.

## How to translate (core) modules

All core modules and many other modules use an online service for keeping translations in sync and up to date over many languages, called www.transifex.com. Module maintainers can upload the source locale and download all translations via CLI tools. When a module uses transifex, that's the source of all translations. Therefor, you should never modify a translation in your git repository, only the main source locale.

For more information please have a look at [Silverstripe's documentation ](https://docs.silverstripe.org/en/5/contributing/translations/)

One nice feature of transifex is suggesting translations, which is very handy if e.g. the key of a translation changed and you don't need to translate that string again. This saves a lot of work and helps you to keep your translations in a consistent style.

## How can we translate JavaScript?

Silverstripe CMS also has a helper for translating strings in JavaScrpt, similar to `_t()` in PHP. All you need to do is to include the translation library and the JavaScript translations into your JS bundle, then you're ready to go.

If you use it in frontend, you need to include Silverstripe's standalone 18n JavaScript library:

```php
use SilverStripe\View\Requirements;

Requirements::javascript('silverstripe/admin:client/dist/js/i18n.js');
Requirements::add_i18n_javascript('<my-module-dir>/javascript/lang');
```
The translations are automatically included depending on i18n's current locale. English is always included as a fallback.

The translations in _javascript/lang/en.js_ look like:

```javascript
if (typeof (ss) === 'undefined' || typeof (ss.i18n) === 'undefined') {
  /* eslint-disable-next-line no-console */
  console.error('Class ss.i18n not defined');
} else {
  ss.i18n.addDictionary('en', {
    'MYMODULE.CATEGORY': 'Kategorie'
  });
}
```
and in an example file for Portuguese _javascript/lang/pt.js_:

```javascript 
ss.i18n.addDictionary('en', {
    'MYMODULE.CATEGORY': 'categoria'
  });
```

In JavaScript you can use the translation like this:

```javascript
ss.i18n._t('MYMODULE.CATEGORY')
```

`ss.i18n` also contains some functions to help with dynamic variables in translations, similar to `_t()` in PHP.


```javascript
// MYMODULE.MYENTITY contains "Added {count} items to cart"
// The myText variable contains: "Added 42 items to cart"
const myText = ss.i18n.inject(
  ss.i18n._t('MYMODULE.MYENTITY'),
  42
);
```

## And the content?

Now we know how to translate static content in your webapp, either in PHP, templates or JavaScript. But what about the content that's stored in the database?

This is part of the next article in this series. 
