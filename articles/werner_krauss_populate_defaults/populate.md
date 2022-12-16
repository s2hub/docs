# Tips and tricks for populating default records with Silverstripe CMS

In Austria, I often hear the proverb "faul, fauler, EDVler", which can be translated as "lazy, lazier, IT-specialaziest". And I tend to be lazy when it comes to adding e.g. predefined site structure or data to my project. While developing I use fake data and lorem ipsum texts to fill the site with content and when deploying I want to start with a clean database. Luckily, Silverstripe has some nice features and modules that help me to avoid adding all that data in the backend manually. Because I'm lazy and don't like clicking around in the backend if I can avoid it. 

One powerful feature is called "default records". This means, when the table for your `DataObject` is created on `dev/build`, it adds some records to the database, and you can start working without adding this data manually. The most popular example are the default pages (home, about us, contact) you find on a blank installation.

## Don't create default pages on first dev/build

While the default pages created by Silverstripe are handy, in some cases you want to start with a clean database and add the pages to your project manually. Then you really don't have any data left and the index starts with no. 1. You can switch them off with the following setting:

```yaml
SilverStripe\CMS\Model\SiteTree:
  create_default_pages: false
```

## Create page structure

In case you want to create a site structure with yml, you can use a module called [silverstripe-populate](https://github.com/silverstripe/silverstripe-populate). You can define fixtures for your model in separate yml files that will be written to the database on dev/build. This is handy, when you want to throw away your development database with mock data and start again from scratch. Or you can define e.g. data for team members in a yml fixture for starting, in case you think that editing yml is easier than using CMS web forms.

To enable the module you need to configure the path to your fixture files.

```yaml
DNADesign\Populate\Populate:
  include_yaml_fixtures:
    - 'app/fixtures/populate.yml'
```

In that fixture file you can add the data for your models. One fixture per database entry:

```yaml
Page:
  home:
    Title: "Home"
    Content: "My Home Page"
    ParentID: 0
  category:
    Title: "A Category"
    ParentID: 0
  subpage1:
    Title: "A subpage of Category"
    ParentID: =>Page.category
```

In this very simple example I didn't use any advanced page types. Note, that "subpage1" references to the other fixture "category" for getting its `ParentID`.

More information can be found in the [very good module documentation](https://github.com/silverstripe/silverstripe-populate#readme). 

By default, silverstripe-populate only runs in dev mode. Be sure to remove the module from composer when the project is live. Otherwise, in some very rare settings combined with an unsecure configurations, the task can be run again by a bot and delete your database. Did I mention, that backups are always a good idea?

## Defining default locales for fluent

I prefer to define as many configurations as possible in the yaml config. This way it lives in my git repository and I don't have to add all the locales again when deleting my sandbox database with mock data to start with live data. Fortunately you can use `DataObject`'s `default_records` to add your locales on dev/build. I often use this configuration:

```yaml
---
Name: myfluentconfig
After: '#fluentconfig'
---

TractorCow\Fluent\Model\Locale:
  default_records:
    de:
      Title: Deutsch
      Locale: de_DE
      URLSegment: de
      IsGlobalDefault: 1
    en:
      Title: English
      Locale: en_GB
      URLSegment: en
```

## Using silverstripe-populate and fluent together

Of course, you can also use silverstripe-populate to define the project's locales if you need to create. In this case your fixtures need also to fill all the `Foo_Localised` tables with the translations. There is no need to add the default locale to the localisation table, as they're added when the record is created before.:

```yaml
TractorCow\Fluent\Model\Locale:
  de:
    Title: Deutsch
    Locale: de_DE
    URLSegment: de
    IsGlobalDefault: 1
  en:
    Title: English
    Locale: en_GB
    URLSegment: en

Page:
  about:
    Title: About Us
    URLSegment: about-us

SiteTree_Localised:
  about_en:
    RecordID: =>Page.about
    Locale: en_GB
    Title: About us
    URLSegment: about-us
```
