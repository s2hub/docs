# How to validate ORM relations in Silverstripe

Sometimes, the defined relations in our model can be incorrect. That's normal and we're just developers. Well, even AI-devs like ChatGPT or Copilot can make mistakes, not just us. That's where all those testing and validation tools are invaluable.

Silverstripe Framework comes with a tool called `RelationValidationService` that checks if the relationships are set up correctly in both directions. It's run on every dev/build. That's good. But unfortunately, you have to configure it a little bit, and then it may shout out a long red list when running dev/build?flush. 

## Common mistakes in relations

Like in real life, even if you think your relations are fine, they could be more perfect. It's always worth improving relations, so what are the common mistakes for Silverstripe ORM relations?

### Relation to an unknown class or relation is no DataObject

Well, this is easy to spot. Dev/build will break and you're forced to fix it.

### Back relation is missing

Common for `has_one` to e.g. `Image`. Technically it works, but for Silverstripe it's better to know what's the relation back, even if you don't use it in your code. Who knows, maybe a future developer (read: you) want's to crate a task that checks if an image is used somewhere or if the image can be deleted, then the relation back can be used. 

### Relation is not in the expected format

Sometimes, Silverstripe expects you to write `ClassName.RelationName`. to fully understand your relations.

### Back relation is ambiguous

The back relations of a `has_many` is a `has_one`. And if, for some reason, the other class has two `has_one`s to the original class, we all get confused, not only Silverstripe`s ORM.

## RelationValidationService's Configuration

In order to switch on `RelationValidationService` on every dev/build, you need to configure the namespaces of the `DataObject`s you want to be validated, and you need to enable output by default.

```yaml
---
Only:
  environment: 'dev'
---

SilverStripe\Dev\Validation\RelationValidationService:
  output_enabled: true
  allow_rules:
    myNamespace: MyCompany
```

Then you see a bunch of errors, can go through your code and adjust the relation namings, e.g. convert

```php
    public static array $has_many = [
        'PhotoGalleryImages' => PhotoGalleryImage::class
    ];
```

to

```php
    public static array $has_many = [
        'PhotoGalleryImages' => PhotoGalleryImage::class. '.Album'
    ];
```

if the relation back is called "Album".

For third party `DataObject`s you refer to, you can use yaml config to add relations back, e.g.:

```yaml
SilverStripe\Assets\Image:
  belongs_to:
    Event: MyProject\Model\Event.Image
    Notification: MyProject\Model\Notification.Image
```

I think this tool should be switched on by default in dev environments. In an ideal world you hook it to your unit tests to get warnings as soon as possible.

[More information about RelationValidationService ](https://docs.silverstripe.org/en/5/developer_guides/model/relations/#validating-relations), e.g. how it can be included in unit tests, can be found in the documentation.