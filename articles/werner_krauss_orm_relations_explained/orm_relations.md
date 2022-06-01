# Relations in Silverstripe's ORM

## Introduction

Life is full of relations. We're born as children of our parents, siblings to our brothers and sisters, have friends and school-mates, and later relations to
business partners or the loved one we marry.
And there are also technical relations between different sets of information we store in our database. Member records,
image galleries, customer data, shop orders, etc. One thing I really love about Silverstripe CMS, is that it helps me to
model these relations with its very good ORM. I can concentrate on the objects I want to model and the framework
does everything needed in the database: creating tables, adding fields and indexes, querying for results etc...

Of course, it can help to know how databases work to understand what's going on under the hood when you use tools like the ORM that assist us
with modelling - but for me, it's good that I don't have to deal with that all the time. This way I can concentrate on the
business logic and finish my projects faster with fewer errors.

All data tables in Silverstripe CMS are defined as subclasses of `DataObject`. Each `DataObject` instance represents a single row in a database table, following the "Active Record" design pattern. (See [documentation: Data Model and ORM](https://docs.silverstripe.org/en/4/developer_guides/model/data_model_and_orm/))

Let's dig into the possible relation types, how they are managed in Silverstripe CMS and how we can extend an existing installation with our own model:

## One to One: exclusive relation

We start with the 1:1 relations (one to one). Its use case is a bit special, and you might not use it too often.

You can use it if you need really exclusive relations. E.g. you want to store additional data for a `Member` outside
the `Member` table, either to keep things seperated or when you hit limits of your database, like maximum column
restrictions. Each member can have exactly one extra data record of the corresponding type, and vice versa.

Another example could be that you want to model e.g. a marriage between two people. Then you have a 1:1 relation between one
database table (e.g. Person <-> Person). Yes, that's possible, too.

### SQL explained: Data Tables; Foreign key

How does your database save that relation? It uses a so called "foreign key" to hold the unique identifier, the "primary
key" of the related record. In Silverstripe CMS that primary key is the `ID` field of our DataObject. In plain SQL it could
be any unique field.

| ID | Created | Title |
----| --- | --- | 
| 1 | 2022-05-22 | This is a test |
| 2 | 2022-05-25 | Another brick in the wall |

Silverstripe's ORM handles the relation for you and adds the relevant field and index to the database automatically. All
you have to do is define the relation you want. Let's start with an example:

### Example: extra data in other DataObjects to keep the main table smaller

Let's say users can log in to your site using OAuth (e.g. login with GitHub) and you want to store some extra data about
that user in a seperate table to keep the tables smaller.
We need to store e.g. Country, the GitHub username and other information. The table should look like this:

|ID | Created | LastEdited | Country | GHUserName | Comment |
----| --- | --- |----| --- | --- | 
42 | 2022-05-15  | 2022-05-16 | AT | wmk | loves clean code |
43 | 2022-05-22 | 2022-05-22 | UK | xyz | is anonymous |

Let's add this table to the database. In Silverstripe CMS all you need to do is define a DataObject:

```php
use SilverStripe\ORM\DataObject;
use SilverStripe\Security\Member; 

class S2HubDocs\Model\MemberData extends DataObject;
{
    private static $table_name = 'MemberData';
    
    private static $db = [
        'Country' => 'Varchar',
        'GHUserName' => 'Varchar',
        'Comment' => 'HTMLText'
    ];
}
```

Silverstripe CMS will add a new table named `MemberData` to your database and add all relevant fields to it when you
run `/dev/build?flush`. The primary key "ID" and the fields for creation and last edited timestamps are added automatically.

**Note:** You can run this command either in the browser by adding it to the base URL of your Silverstripe CMS installation (e.g. http://mysite.test/dev/build?flush) or in CLI by running `vendor/bin/sake dev/build flush` in the base directory of your installation.

The `Member` DataObject holds the foreign key to `MemberData`. We use a [`DataExtension`](https://docs.silverstripe.org/en/4/developer_guides/model/extending_dataobjects/) to add this to
Silverstripe's `Member` class, as we should not edit 3rd party (vendor) classes directly:

```php
use S2HubDocs\Model\MemberData;
use SilverStripe\ORM\DataObject; 

class S2HubDocs\Extension\Member extends DataExtension
{
    private static $has_one = [
        'ExtraMemberData' => MemberData::class
    ];
}
```

With the `$has_one` config array you can define which foreign keys to other tables should be saved in your model's
table. It generates an integer column suffixed with "ID", so our relation name "ExtraMemberData" will create the field "
ExtraMemberDataID". Additionally, Silverstripe CMS will add an index to this table for faster database queries.

Let's plug this `DataExtension` to `Member` by adding the following lines to your [yml config](https://docs.silverstripe.org/en/4/developer_guides/configuration/configuration/):

```yml
Member:
  extensions:
    MemberData: S2HubDocs\Extension\Member
```

Note: if you only want to add the relation and don't need to update CMSFields, you can define the relation in your yml
config, instead.

We should also add the corresponding relation to our `MemberData` class:

```php
class S2HubDocs\Model\MemberData extends DataObject;
{
    ... 
    
    private static $belongs_to = [
        'Member' => Member::class
    ];   
}
```

By adding the config `$belongs_to` to our `MemberData` class, Silverstripe CMS knows we have a 1:1 relation. Theoretically
you don't need it to work, as it adds nothing to the database. It's biggest advantage is, that it's more verbose and you
can use ORM's relation methods to get the related DataObject from both directions.

After running `dev/build/?flush` we can see, that the `Member` table has a new database field "ExtraMemberDataID" that
can hold the foreign key to the related table.

Our Member table now looks like:

| ID | FirstName | Surname | Email | ExtraMemberDataID |
--- | --- | --- | --- | --- | 
|123 | Werner | Krauß | werner.krauss@s2-hub.com | 42

Note, that the Member's ID and the ExtraMemberDataID don't need to be the same number. Each relation is numbered independently.

In our code we can now use magic getter methods named like our relation name; This getter method returns the
corresponding `DataObject`. In our example you can use it like:

```php
$member = Member::get()->byID(123); // Werner Krauß
echo $member->ExtraMemberDataID; //show the ID of the related record - 42
$extraData = $member->ExtraMemberData(); //get the related MemberData object
echo $extraData->Comment; //echo the saved comment for this member - "loves clean code"
```

We assume for this example that Member no. 123 has related MemberData and therefor we don't check for its existence.

The same works vice versa, because we added `$belongs_to` to the other side of the relation. Remember, that the foreign
key is saved at the Member object, so we first need to get that with the magic relation method:

```php
$memberData = MemberData::get()->byID(42);
echo $memberData->Member()->FirstName; // show the FirstName field of the related Member - Werner
```

### Example: Marriage: 1:1 between person objects

It's also possible to add relations to the same object. Let's create a `Person` Model and create relations between
persons:

```php
use SilverStripe\ORM\DataObject;

class Person extends DataObject
{
    private static $table_name = 'Person';
    
    private static $db = [
        'FirstName' => 'Varchar',
        'Surname' => 'Varchar',
        'Birthday' => 'Date'
    ];
    
    private static $has_one = [
        'MarriedTo' => Person::class,
    ];
    
    private static $belongs_to = [
        'IsMarriedTo' => Person::class,
    ];
    
    public function getIsMarried(): bool
    {
        return $this->MarriedTo()->exists() || $this->IsMarriedTo()->exists;
    }

}
```

After creating the database table by running `dev/build/?flush` we can add some data to it:

```php
$mary = Person::create();
$mary->FirstName = 'Mary';
$mary->Surname = 'Doe';
$mary->write();

$john = Person::create();
$john->FirstName = 'John';
$john->Surname = 'Doe';
$john->MarriedToID = $mary->ID;
$john->write();

$johnIsMarriedTo = $john->MarriedTo()->FirstName;
$maryIsMarriedTo = $mary->IsMarriedTo()->FirstName;
```

Our Person table now looks like this:

| ID  | FirstName | Surname | Birthday | MariedToID |
|-----| --- | --- | --- | --- |
| 1 | John | Doe | 1999-01-01 | 2 |
| 2 | Mary  | Doe | 2000-02-02 | | 

> Note: a 1:1 relation to the same object needs more logic to be bullet proof. In the example above nothing holds us from setting Mary's `MarriedToID` to someone else's ID. This can be handled e.g. in `onAfterWrite()` and `delete()` when a record is removed.

> Tip: if you want to inline your 1:1 relation's fields you can use a module called [Has One Edit](https://packagist.org/packages/stevie-mayhew/hasoneedit). It handles saving into the has_one relation for you automatically. 
> This way the data in your user interface feels like one model, though it's saved over two or more tables.
> 
> Since Silverstripe CMS [4.9](https://docs.silverstripe.org/en/4/changelogs/4.9.0/#other-new-features) is is possible to name form fields with dot notation to achieve this without a module. E.g. `\SilverStripe\Forms\TextField::create('ExtraMemberData.GHUserName', 'GitHub user name')`

## One to Many: Hierarchy, trees and taxonomies

While the 1:1 relation is neat, you cannot model everything with it. In fact, it's very limited. What if you want an
image gallery with albums? Then one album has more than one image. This is where the 1:N (one to many) relation comes
in.

### Example: a very simple image gallery

1:N uses `$has_many` on the "1" side (our album) and `$has_one` on the "many" side (the images). This means that an
album can have many images and each image is attached to exactly one album. If you want your images to be shown in more
albums, wait a bit until we discuss many-to-many relations.

```php
use SilverStripe\ORM\DataObject;
use SilverStripe\Assets\Image;

class S2HubDocs\Model\Album extends DataObject
{
    private static $table_name = 'Album';
    
    private static $db = [
        'Title' => 'Varchar'
    ];
    
    private static $has_many = [
        'Images' => Image::class;
    ];
}
```

Let's add the corresponding `$has_one` in our yml config to avoid editing vendor code:

```yml
Image:
  has_one:
    Album: S2HubDocs\Model\Album
```

After running `dev/build/?flush` we have a new table "Album" and Image has a new column for the foreign key, called "AlbumID". We can get all
images of an album by calling the magic getter method, which returns a HasManyList, a subclass of DataList with some
extra methods for managing 1:N relations.

```php
$myAlbum = Album::get()->byID(27);
$numberOfImages = $myAlbum->Images()->count(); //how many images are in this album?

$image = Image::get()->byID(42);
$album = $Image->Album(); //get the album of an image
```

In the example above, `$myAlbum->Images()` returns a `HasManyList`, which is a subclass of `RelationList` and `DataList`.
Those lists are used to handle relation specific tasks like adding or removing objects from a relation (see below).

### Example: parent/child relation

Another example for a typical 1:N relation is the parent/child relationship. A child has exactly one mother or father, but
each adult can have 0, 1 or many children. This way we can model hierarchy trees by objects relating to each other.
Let's modify the `Person` model from the 1:1 marriage example:

```php
use SilverStripe\ORM\DataObject;

class Person extends DataObject 
{
    ...
    
    private static $has_one = [
        'MarriedTo' => Person::class,
        'Parent' => Person::class . '.Children',
    ];
    
    private static $has_many = [
        'Children' => Person::class . '.Parent',
    ];
}
```

Note, that we have a 1:N relation to the same DataObject, therefor we have to use named relations (see [Documentation](https://docs.silverstripe.org/en/4/developer_guides/model/relations/))

### Adding objects to RelationLists

While you can save the foreign key manually to your DataObject on the "N" side of a 1:N relation, Silverstripe CMS gives you a subclass of
RelationList (e.g. HasManyList or ManyManyList), which has a handy way to add objects to an existing relation:

```php
$sarah = Person::get()->byID(11);
$john->Children()->add($sarah); 
```

The [`->add()` method](https://api.silverstripe.org/4/SilverStripe/ORM/HasManyList.html#method_add) takes a DataObject, adds the foreign key for us and saves it in one take. I think, the code of that is pretty elegant...

The more verbose version of the code above would be:

```php
$sarah = Person::get()->byID(11);
$sarah->ParentID = $john->ID;
$sarah->write();
```
Our table now looks like this:

| ID  | FirstName | Surname | Birthday | MariedToID | ParentID |
|-----|-----------| --- | --- | --- | --- |
| 1   | John      | Doe | 1999-01-01 | 2 | |
| 2   | Mary      | Doe | 2000-02-02 | | |
| 2   | Mary      | Doe | 2000-02-02 | | |
| 11  | Sarah     | Doe | 2010-10-10 | | 1| 


In case you want to remove an item from a list, you can use the `->remove()` method. Be careful, this might delete the dataset from your database, depending if you're using a plain `DataList` (`Person::get()->filter('ParentID', $john->ID)`) or a `RelationList` (`$john->Children()`), where it removes the relationship between the records instead.

```php
//I cannot imagine why someone would like to remove a child;
//must have been a wrong relation in first place.
$john->Children()->remove($sara);
// Sarah still exists in our database, but no longer has John as a parent
```

#### Excourse: Hierarchy extension

Modelling tree-like data is pretty common, and Silverstripe CMS offers a handy tool so you don't have to reinvent the wheel
all the time:
the [hierarchy extension](https://github.com/silverstripe/silverstripe-framework/blob/4/src/ORM/Hierarchy/Hierarchy.php). 
It adds the `Parent()` and `Children()` relations for you, can work with Versioned extension (for unpublished and published
versions of the same object), offers methods for getting all descendant objects, takes permissions into account, etc...
The most obvious example of the `Hierarchy` extension in action is `SiteTree` - the primary functionality of the CMS.

## Many to Many:

The M:N relation (many to many) is the only relation that cannot be directly established between two database tables. We
need an extra mapping table in the middle that resolves the M:N relation into two 1:N relations and holds the foreign keys of both sides (also known as a "join table").
Luckily Silverstripe CMS does that for you, and you don't have to think about that when modelling the database.

### Basic relation

All you need is to define `$many_many` _and_ `$belongs_many_many` on both ends of the relation. If you have e.g. blog
posts you want to be tagged, each blog post can have many tags, and each tag can have many blog posts. A typical use
case of M:N relations might be:

```php
class BlogPost extends Page 
{
    ...
    
    private static $many_many = [
        'Tags' => Tag::class
    ];
}
```

```php
class Tag extends DataObject
{
    private static $db = [
        'Title' => 'Varchar'
    ];
    
    private  static $belongs_many_many = [
        'BlogPosts' => BlogPost::class
    ];
}
```

> A common convention in Silverstripe CMS development is to name `$has_many`, `$many_many` and `$belongs_many_many` relations in plural. This ensures that the code is more readable. The relation holding many tags is named "Tags" and the corresponding DataObject is named in singular "Tag", as each object only holds data for one specific tag.

The code above will create a mapping table called "BlogPost_Tags", the combined name of the M and the N side of the M:N relation. It holds an ID (for convenience) and both foreign keys for BlogPosts and Tags:

ID | BlogPostID | TagID |
--- |--- |--- | 
1 | 123  |4 |
2 | 123 | 12 |
3 | 144 | 4 |
4 | 128 | 12 |

Any given combination between BlogPost and Tag is possible with a many-many relation.

As with the 1:N example above, we can access the relation with the ORM's magic methods:

```php
$blogPost = BlogPost::get()->byID(77);
$tags = $blogPost->Tags(); //returns a ManyManyList
$numberOfTags = $tags->count();
```

This time we get an object of the type `ManyManyList`, another subclass of `DataList` with special methods for handling the M:N relation. Adding or removing objects is similar with `has_many` above.

### Add some extra fields...

Currently, we only have the relation between two DataObjects enabled. When we query the database the objects can be returned in any order. To reorder objects we need to add extra data to the mapping table.

Let's say our `Person` from above will have `Hobbies`. Each person can have many different hobbies, and each hobby is done by many different people. And you should be able to weight the hobbies per person, e.g. by adding points from 1 to 100 - to indicate their favourite hobbies compared to others they have. This weight field will be added to the  manymany junction table, because it's data that is relevant _this relationship_ between the person and his or her hobby, rather than an attribute of the person or the hobby specifically. You can tell Silverstripe CMS to do that by defining `$many_many_extraFields`.

```php
class Person extends DataObject
{
    ...
    
    private static $many_many = [
        'Hobbies' => Hobby:class
    ];
  
    private static $many_many_extraFields = [
        'Hobbies' => [
            'Weight' => 'Int'
        ]
    ];
}
```

```php
class Hobby extends DataObject
{
    private static $table_name = 'Hobby';
    
    private static $db = [
        'Title' => 'Varchar'
    ];
    
    private static $belongs_many_many =  [
        'Persons' => Person::class
    ];
}
```

After creating the relation by running `dev/build/?flush`, let's add John's favourite hobbies:

```php
$playingTheGuitar = Hobby::get()->filter(['Title' => 'Playing the guitar']);
$running = Hobby::get()->filter(['Title' => 'Running']);

$john->Hobbies()->add($playingTheGuitar, ['Weight' => 100]);
$john->Hobbies()->add($running, ['Weight' => 75]);

$mary->Hobbies()->add($running, ['Weight' => 85]); // Mary likes running more than John
```

Our Person_Hobbies mapping table now looks like this:

ID | PersonID | HobbyID | Weight |
--- |--- |--- |--- |
1  | 1 | 6 | 100 |
2 | 1 | 21 | 75 |
3  | 2 | 21 | 85 | 


### Handling M:N with an extra DataObject: $many_many_through

If you need extra logic in the mapping table, e.g. some `onBeforeWrite()` magic or extensions like Fluent or Versioning, you can define a specific `DataObject` for mapping in `$many_many_through`.  Our person - hobbies relation could look like this:

```php
class PersonHobbyMapping extends DataObject
{
    private static $table_name = 'PersonHobbyMapping';
    
    private static $db = [
        'Weight' => 'Int'
    ];
    
    private static $has_one = [
        'Person' => Person::class,
        'Hobby' => Hobby::class
    ];
}
```


```php
class Person extends DataObject
{
    ...
    
    private static $many_many = [
        'Hobbies' => [
            'through' => PersonHobbyMapping::class,
            'from' => 'Person',
            'to' => 'Hobby'
        ]
    ];
  
    private static $many_many_extraFields = [
        'Hobbies' => [
            'Weight' => 'Int'
        ]
    ];
}
```

On `Hobby`, `$belongs_many_many` stays the same. We still get a `ManyManyList` and adding or removing relations from Person to Hobby (or vice versa) is the same as above. The advantage of this is that we can query `PersonHobbyMapping` directly, if we need to. The setup you use is specific to the needs of your project.

## Conclusion

One of Silverstripe's key features is that you can model your data structure in PHP classes, and all database tables are created automatically by running `dev/build/?flush`. This helps you to get your work done faster. It's good to know how the relations work in the database, but it's less error-prone if I don't have to create or migrate the tables myself.

While Silverstripe CMS updates your tables automatically, there is no ready-to-use system for handling migrations or downgrades. It doesn't delete columns you don't need any more, because it doesn't know if you still need the data or if it should get rid of it. This isn't a problem for me. I tend to write migration documentation with SQL statements I can run manually.

**Warning:** Make sure to delete unused columns manually, because this might lead to unexpected behaviour, if you reuse an old column with existing data in a long-running project. 

What's next? The backend in Silverstripe CMS offers e.g. `ModelAdmin` for managing your models and `GridField` for managing relations. But that's content for a different tutorial...

## Sources:

* [Silverstripe Documentation: Data Model and ORM](https://docs.silverstripe.org/en/4/developer_guides/model/data_model_and_orm/)
* [Silverstripe Documentation: Relations](https://docs.silverstripe.org/en/4/developer_guides/model/relations/)
