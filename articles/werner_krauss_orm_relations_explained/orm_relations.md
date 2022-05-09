# Relations in Silverstripe ORM

## Introduction

Life is full of relations. We're born as siblings of our parents, have friends and school-mates, and later relations to business partners or the loved one we marry.
And there are that technical relations between different sets of informations we store in our database. Member records, image galleries, customer data, shop orders, etc. One thing I really love about Silverstripe is, that it helps me to model these relations with its very good ORM. The framework helps me to concentrate on the objects I want to model and does everything needed in the database. Creating tables, adding fields and indexes, querying for results etc... 

Of course, you should know how databases work to understand what's going on under the hood when you use tools that help you with the model. For me it's good that I don't have to deal with that all the time. This way I can concentrate on the business logic and finish my projects faster with less errors. 

Let's dig into the possible relation types and how they are managed in Silverstripe:

##  One to One: exclusive relation

We start with the 1:1 relations (one to one). Its use case is a bit special and you might not use it too often.

You can use it if you need really exclusive relations. E.g. you want to store additional data for a `Member` outside the `Member`table, either to keep things seperated or when you hit limits of your database, like maximum column restrictions. Each member can have exactly one extra data record and vice versa.

Another example could be that you want to model e.g. marriage between persons. Then you have a 1:1 relation between one database table. Yes, that's possible, too.

### SQL explained: Data Tables; Foreign key
How does your database save that relation? It uses a so called "foreign key" to hold the unique identifier, the "primary key" of the related record. In Silverstripe that primary key is the `ID` field of our DataObject. In plain SQL it could be any unique field.

** TODO: example tables ** 

Silverstripe's ORM handles the relation for you and adds the relevant field and index to the database automatically. All you have to do is defining the relation you want. Let's start with an example:

### example: extra data in other DO to keep main table smaller
Let's say users can log in to your site using OAuth (e.g. login with GitHub) and you want to store some extra data about that user in a seperate table to keep the tables smaller.
We need to store e.g. Country, the GitHub username and other information. The table should look like this:

|ID   | Created | LastEdited | Country   | GHUserName   | Comment   |

Let's add this table to the database. In Silverstripe all you need to do is creating a DataObject:

```php
use DataObject;
use Member; //@todo: correct Namespaces
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

Silverstripe will add a new table named `MemberData` to your database and add all relevant fields to it when you run `/dev/build?flush`. The primary key "ID" and the fields for creation and last edited timestamps are added automatically.

The `Member` DataObject holds the foreign key to `MemberData`. We use a `DataExtension` to add this to Silverstripe's `Member` class, as we should not edit 3rd party classes directly:

```php
use S2HubDocs\Model\MemberData;
use DataObject; //@todo: correct namespace

class S2HubDocs\Extension\Member extends DataExtension
{
    private static $has_one = [
        'ExtraMemberData' => MemberData::class
    ];
}
```
//TODO: verify index on relations.

With the `$has_one` config array you can define, which foreign keys to other tables should be saved in your model's table. It generates an integer column suffixed with "ID", so our relation name "ExtraMemberData" will create the field "ExtraMemberDataID". Additionally, Silverstripe will add an index to this table for faster database queries.

Let's plug this `DataExtension` to `Member` by adding the following lines to your yml config:

```yml
Member:
  extensions:
    MemberData: S2HubDocs\Extension\Member
```

Note: if you only want to add the relation and don't need to update CMSFields, you can add the relation in your yml config, too.

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

By adding the config `$belongs_to` to our `MemberData` class, Silverstripe knows we have a 1:1 relation. Theoretically you don't need it to work, as it adds nothing to the database. It's biggest advantage is, that it's more verbose and you can use ORM's relation methods to get the related DataObject from both directions. 

After running `dev/build/?flush` we can see, that the `Member` table has a new database field "ExtraMemberDataID" that can hold the foreign key to the related table.

In our code we can now use magic getter methods named like our relation name; This getter method returns the corresponding `DataObject`. In our example you can use it like: 

```php
$member = Member::get()->byID(123);
$echo $member->ExtraMemberDataID; //show the ID of the related record.
$extraData = $member->ExtraMemberData(); //get the related MemberData object
echo $extraData->Comment; //echo the saved comment for this member
```

We assume for this example that Member no. 123 has related MemberData and therefor we don't check for its existence.

The same works vice versa, because we added `$belongs_to` to the other side of the relation. Remember, that the foreign key is saved at the Member object, so we first need to get that with the magic relation method:  

```php
$memberData = MemberData::get()->byID(42);
echo $memberData->Member()->ID; // show the ID of the related Member 
```

### example: marriage: 1:1 between person objects
It's also possible to add relations to the same object. Let's create a `Person` Model and create relations between persons:

```php
use DataObject; //@todo: correct namespaces
class Person extends DatabaseObject
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

#### Excursus: edit has one module (ex simon welsh)

//TODO: write about has_one_edit module

## One to Many: Hierarchy, trees and taxonomies

While the 1:1 relation is neat, you cannot model everything with it. In fact it's very limited. What if you want an image gallery with albums? Then one album has more than one image. This is where the 1:n (one to many) relation comes in.

### example: simple image gallery

1:n uses `$has_many` on the "1" side (our album) and `$has_one` on the "many" side (the images). This means that an album can have many images and each image is attached to exactly one album. If you want your images been shown in more albums, wait a bit until we discuss many-to-many relations.

```php
//@todo: namespaces...
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

After running `dev/build/?flush` we have a new table "Album" and Image has a new column called "AlbumID". We can get all images of an album by calling the magic getter method, which returns a HasManyList, a subclass of DataList with some extra methods for managing 1:n relations.

```php
$myAlbum = Album::get()->byID(27);
$numberOfImages = $myAlbum->Images()->count(); //how many images are in this album?

$image = Image::get()->byID(42);
$album = $Image->Album(); //get the album of an image
```

In the example above, `$myAlbum->Images()` returns a `HasManyList`, which is a subclass of `RelationList` and `DataList`. Those lists are used to handle relation specific tasks like adding or removing objects from a relation (see below).

### example: parent/child relation

Another example for a typical 1:n relation is the parent child relation. A child has exactly one mother or father, but each adult can have 0, 1 or many children. This way we can model hierarchy trees by objects relating to each other. Let's modify the `Person` model from the 1:1 example:

```php
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

Note, that we have a 1:n relation to the same DataObject, therefor we have to use named relations (see Documentation)

//TODO: link to Documentation
//todo: example data table

### Adding objects to RelationLists

While you can save the foreign key manually to your DataObject on the "n" side of a 1:n relation, Silverstripe's RelationLists (e.g. HasManyList or ManyManyList) have a handy way to add objects to an existing relation:

```php
$juli = Person::get()->byID(11);
$john->Children()->add($juli); 
```

The `->add()` method takes a DataObject, adds the foreign key for us and saves it in one take. IMHO the code is pretty elegant... 

//TODO: link to code of add method

The more verbose version of the code above would be:
```php
$juli = Person::get()->byID(11);
$juli->ParentID = $john->ID;
$july->write();
```

//@todo: more handy methods like removing an item from a list, etc..?
#### excourse: hierarchy extension

Modelling tree-like data is pretty common, and Silverstripe offers a handy tool so you don't have to reinvent the wheel all the time: The [hierarchy extension](https://github.com/silverstripe/silverstripe-framework/blob/4/src/ORM/Hierarchy/Hierarchy.php). It adds the Parent and Children relations for you, can work with Versioned extension (for unpublished and published versions of the same object), offers methods for getting all descendant objects, takes permissions into account, etc... The most obvious example of this extension is SiteTree.

## Many to Many: 

The m:n relation (many to many) is the only relation that cannot be directly established between two database tables. We need an extra table in the middle that resolves the m:n relation into two 1:n relations and holds the foreign keys. Luckily Silverstripe does that for you and you don't have to think about that when modelling the database.

* Returns ManyManyList
* $manymany / $belongs_manymany
### classic relation
### manymany through


### Example: `->relation()`

## Excurse: SS_List interface
### why
### explain available Subclasses
### extra logic in own subclass (e.g. in Silvershop)

Sources:
* [Silvesrstripe Documentation: Relations](https://docs.silverstripe.org/en/4/developer_guides/model/relations/)