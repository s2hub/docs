# Relations in Silverstripe ORM

## Introduction

Life is full of relations. We're born as siblings of our parents, have friends and school-mates, and later relations to
business partners or the loved one we marry.
And there are that technical relations between different sets of informations we store in our database. Member records,
image galleries, customer data, shop orders, etc. One thing I really love about Silverstripe is, that it helps me to
model these relations with its very good ORM. I can concentrate on the objects I want to model and the framework
does everything needed in the database: creating tables, adding fields and indexes, querying for results etc...

Of course, you should know how databases work to understand what's going on under the hood when you use tools that help
you with the model. For me it's good that I don't have to deal with that all the time. This way I can concentrate on the
business logic and finish my projects faster with less errors.

Let's dig into the possible relation types and how they are managed in Silverstripe:

## One to One: exclusive relation

We start with the 1:1 relations (one to one). Its use case is a bit special and you might not use it too often.

You can use it if you need really exclusive relations. E.g. you want to store additional data for a `Member` outside
the `Member`table, either to keep things seperated or when you hit limits of your database, like maximum column
restrictions. Each member can have exactly one extra data record and vice versa.

Another example could be that you want to model e.g. marriage between persons. Then you have a 1:1 relation between one
database table. Yes, that's possible, too.

### SQL explained: Data Tables; Foreign key

How does your database save that relation? It uses a so called "foreign key" to hold the unique identifier, the "primary
key" of the related record. In Silverstripe that primary key is the `ID` field of our DataObject. In plain SQL it could
be any unique field.

| ID | Created | Title |
----| --- | --- | 
| 1 | 2022-05-22 | This is a test |
| 2 | 2022-05-25 | Another brick in the wall |

Silverstripe's ORM handles the relation for you and adds the relevant field and index to the database automatically. All
you have to do is defining the relation you want. Let's start with an example:

### example: extra data in other DO to keep main table smaller

Let's say users can log in to your site using OAuth (e.g. login with GitHub) and you want to store some extra data about
that user in a seperate table to keep the tables smaller.
We need to store e.g. Country, the GitHub username and other information. The table should look like this:

|ID | Created | LastEdited | Country | GHUserName | Comment |
----| --- | --- |----| --- | --- | 
42 | 2022-05-15  | 2022-05-16 | AT | wmk | loves clean code |
43 | 2022-05-22 | 2022-05-22 | UK | xyz | is anonymous |

Let's add this table to the database. In Silverstripe all you need to do is creating a DataObject:

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

Silverstripe will add a new table named `MemberData` to your database and add all relevant fields to it when you
run `/dev/build?flush`. The primary key "ID" and the fields for creation and last edited timestamps are added
automatically.

The `Member` DataObject holds the foreign key to `MemberData`. We use a `DataExtension` to add this to
Silverstripe's `Member` class, as we should not edit 3rd party classes directly:

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

With the `$has_one` config array you can define, which foreign keys to other tables should be saved in your model's
table. It generates an integer column suffixed with "ID", so our relation name "ExtraMemberData" will create the field "
ExtraMemberDataID". Additionally, Silverstripe will add an index to this table for faster database queries.

Let's plug this `DataExtension` to `Member` by adding the following lines to your yml config:

```yml
Member:
  extensions:
    MemberData: S2HubDocs\Extension\Member
```

Note: if you only want to add the relation and don't need to update CMSFields, you can add the relation in your yml
config, too.

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

By adding the config `$belongs_to` to our `MemberData` class, Silverstripe knows we have a 1:1 relation. Theoretically
you don't need it to work, as it adds nothing to the database. It's biggest advantage is, that it's more verbose and you
can use ORM's relation methods to get the related DataObject from both directions.

After running `dev/build/?flush` we can see, that the `Member` table has a new database field "ExtraMemberDataID" that
can hold the foreign key to the related table.

Our Member table now looks like:

| ID | FirstName | Surname | Email | ExtraMemberDataID |
--- | --- | --- | --- | --- | 
|123 | Werner | Krauß | werner.krauss@s2-hub.com | 42

Note, that the Member's ID and the ExtraMemberDataId don't need to be the same number. Each relation is numbered independently.

In our code we can now use magic getter methods named like our relation name; This getter method returns the
corresponding `DataObject`. In our example you can use it like:

```php
$member = Member::get()->byID(123);
$echo $member->ExtraMemberDataID; //show the ID of the related record.
$extraData = $member->ExtraMemberData(); //get the related MemberData object
echo $extraData->Comment; //echo the saved comment for this member
```

We assume for this example that Member no. 123 has related MemberData and therefor we don't check for its existence.

The same works vice versa, because we added `$belongs_to` to the other side of the relation. Remember, that the foreign
key is saved at the Member object, so we first need to get that with the magic relation method:

```php
$memberData = MemberData::get()->byID(42);
echo $memberData->Member()->FirstName; // show the FirstName field of the related Member 
```

### example: marriage: 1:1 between person objects

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
    
    public function getIsMarried {
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


> Tip: if you want to inline your 1:1 relation's fields you can use a module called [Has One Edit](https://packagist.org/packages/stevie-mayhew/hasoneedit). It handles saving into the has_one relation for you automatically. 
> This way the data in your user interface feels like one model, though it's saved over two or more tables. 

## One to Many: Hierarchy, trees and taxonomies

While the 1:1 relation is neat, you cannot model everything with it. In fact, it's very limited. What if you want an
image gallery with albums? Then one album has more than one image. This is where the 1:n (one to many) relation comes
in.

### example: simple image gallery

1:n uses `$has_many` on the "1" side (our album) and `$has_one` on the "many" side (the images). This means that an
album can have many images and each image is attached to exactly one album. If you want your images been shown in more
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

Let's add the corresponding `$has_one` in our yml config:

```yml
Image:
  has_one:
    Album: S2HubDocs\Model\Album
```

After running `dev/build/?flush` we have a new table "Album" and Image has a new column for the foreign key, called "AlbumID". We can get all
images of an album by calling the magic getter method, which returns a HasManyList, a subclass of DataList with some
extra methods for managing 1:n relations.

```php
$myAlbum = Album::get()->byID(27);
$numberOfImages = $myAlbum->Images()->count(); //how many images are in this album?

$image = Image::get()->byID(42);
$album = $Image->Album(); //get the album of an image
```

In the example above, `$myAlbum->Images()` returns a `HasManyList`, which is a subclass of `RelationList` and `DataList`
. Those lists are used to handle relation specific tasks like adding or removing objects from a relation (see below).

### example: parent/child relation

Another example for a typical 1:n relation is the parent child relation. A child has exactly one mother or father, but
each adult can have 0, 1 or many children. This way we can model hierarchy trees by objects relating to each other.
Let's modify the `Person` model from the 1:1 example:

```php
use SilverStripe\ORM\DataObject;

class Person extends DataObject 
{
    ...
    
    private static $has_one = [
        'MarriedTo' => Person::class,
        'Parent' => Person::class . 'Children',
    ];
    
    private static $has_many = [
        'Children' => Person::class . 'Parent',
    ];
}
```

Note, that we have a 1:n relation to the same DataObject, therefor we have to use named relations (see [Documentation](https://docs.silverstripe.org/en/4/developer_guides/model/relations/))

### Adding objects to RelationLists

While you can save the foreign key manually to your DataObject on the "n" side of a 1:n relation, Silverstripe's
RelationLists (e.g. HasManyList or ManyManyList) have a handy way to add objects to an existing relation:

```php
$juli = Person::get()->byID(11);
$john->Children()->add($juli); 
```

The [`->add()` method](https://api.silverstripe.org/4/SilverStripe/ORM/HasManyList.html#method_add) takes a DataObject, adds the foreign key for us and saves it in one take. I think, the code of that is pretty elegant...

The more verbose version of the code above would be:

```php
$juli = Person::get()->byID(11);
$juli->ParentID = $john->ID;
$july->write();
```
Our table now looks like this:

| ID  | FirstName | Surname | Birthday | MariedToID | ParentID |
|-----| --- | --- | --- | --- | --- |
| 1   | John | Doe | 1999-01-01 | 2 | |
| 2   | Mary  | Doe | 2000-02-02 | | |
| 11  | Juli | Doe | 2010-10-10 | | 1| 


In case you want to remove an item from a list, you can use the `->remove()` method. Be careful, this might delete the dataset from your database, depending if you're on a plain `DataList` or a `RelationList`, where it removes the relation between the records.

```php
//I cannot imagine why someone would like to remove a child;
//must have been a wrong relation in first place.
$john->Children()->remove($juli); 
```

#### excourse: hierarchy extension

Modelling tree-like data is pretty common, and Silverstripe offers a handy tool so you don't have to reinvent the wheel
all the time:
the [hierarchy extension](https://github.com/silverstripe/silverstripe-framework/blob/4/src/ORM/Hierarchy/Hierarchy.php). 
It adds the `Parent()` and `Children()` relations for you, can work with Versioned extension (for unpublished and published
versions of the same object), offers methods for getting all descendant objects, takes permissions into account, etc...
The most obvious example of the `Hierarchy` extension is SiteTree.

## Many to Many:

The m:n relation (many to many) is the only relation that cannot be directly established between two database tables. We
need an extra mapping table in the middle that resolves the m:n relation into two 1:n relations and holds the foreign keys of both sides.
Luckily Silverstripe does that for you and you don't have to think about that when modelling the database.

### classic relation

All you need is to define `$many_many` and `$belongs_many_many` on both ends of the relation. If you have e.g. blog
posts you want to be tagged, each blog post can have many tags, and each tag can have many blog posts. A typical use
case of m:n relations might be:

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

> A common convention in Silverstripe is, to name `$has_many`, `$many_many` and `$belongs_many_many` relations in plural. This ensures that the code is more readable. The relation holding all tags is named "Tags" and the corresponding DataObject is named in singular "Tag", as each object only holds one specific tag.

The code above will create a mapping table called "BlogPost_Tags", the combined name of the m and the n side of the m:n relation. It holds an ID (for convenience) and both foreign keys for BlogPosts and Tags:

ID | BlogPostID | TagID |
--- |--- |--- | 
1 | 123  |4 |
2 | 123 | 12 |
3 | 144 | 4 |
4 | 128 | 12 |

Any given combination between BlogPost and Tag is possible with a many-many relation.

As with the 1:n example above, we can access the relation with the ORM's magic methods:

```php
$blogPost = BlogPost::get()->byID(77);
$tags = $blogPost->Tags(); //returns a ManyManyList
$numberOfTags = $tags->count();
```

This time we get an object of the type `ManyManyList`, another subclass of `DataList` with special methods for handling the m:n relation. Adding or removing objects is similar with `has_many` above. 

### Add some extra fields...

Currently, we only have the relation between two DataObjects enabled. When we query the database the objects can be returned in any order. To reorder objects we need to add extra data to the mapping table.

Let's say our `Person` from above will have `Hobbies`. Each person can have different hobbies, and each hobby is done by different persons. And you should be able to weight the hobbies per person, e.g. by adding points from 1 to 100. This weight field will be added to the  manymany junction table, cause it's a special field for this relation between this specific person and his (or her) hobby. You can tell Silverstripe to do that by defining `$many_many_extraFields`.

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

$mary->Hobbies()->add($running, ['Weight' => 85]);
```

Our Person_Hobbies mapping table now looks like this:

ID | PersonID | HobbyID | Weight |
--- |--- |--- |--- |
1  | 1 | 6 | 100 |
2 | 1 | 21 | 75 |
3  | 2 | 21 | 85 | 


### manymany through

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

On `Hobby`, `$belongs_many_many` stays the same. We still get a `ManyManyList` and adding or removing relations from Person to Hobby (or vice versa) is the same as above.

## Conclusion

One of Silverstripe's key features is, that you can model your data structure in PHP classes and all database tables are created automatically by running `dev/build/?flush`. This helps you to get your work done faster. It's good to know how the relations work in the database, but it's less error-prone if I don't have to create or migrate the tables myself.

While Silverstripe updates your tables automatically, there is no ready-to-use system for handling migrations or downgrades. Silverstripe doesn't delete columns you don't need any more, because it doesn't know if you still need the data or if it should get rid of it. This isn't a problem for me. I tend to write migration documentation with SQL statements I can run manually.

What's next? Silverstripe's backend offers e.g. `ModelAdmin` for managing your models and `GridField` for managing relations. But that's content for a different tutorial...

## Sources:

* [Silvesrstripe Documentation: Relations](https://docs.silverstripe.org/en/4/developer_guides/model/relations/)