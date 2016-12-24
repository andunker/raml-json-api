# raml-json-api
RAML-JSON-API PHP-code generator (based on RAML-types) for Laravel framework, with complete support of JSON-API data format 

JSON API support turned on by default - see `Turn off JSON API support` section bellow 

### Installation via composer:
``` 
composer require rjapi/raml-json-api 
```

### Laravel specific configuration

Add command to ```$commands``` array in ```app/Console/Kernel.php```
```php
protected $commands = [
    RJApiGenerator::class,
];
```

Run in console:
```
php artisan raml:generate raml/articles.raml
```

```raml/articles.raml``` - raml file in raml directory in the root of Your project, 
which should be prepared before or You may wish to just try by copying an example from ``` tests/functional/rubric.raml```

The output will look something like this:
```
Created : /srv/projects_root/Modules/V2/start.php
Created : /srv/projects_root/Modules/V2/Http/routes.php
Created : /srv/projects_root/Modules/V2/module.json
Created : /srv/projects_root/Modules/V2/Resources/views/index.blade.php
Created : /srv/projects_root/Modules/V2/Resources/views/layouts/master.blade.php
Created : /srv/projects_root/Modules/V2/Config/config.php
Created : /srv/projects_root/Modules/V2/composer.json
Created : /srv/projects_root/Modules/V2/Database/Seeders/V2DatabaseSeeder.php
Created : /srv/projects_root/Modules/V2/Providers/V2ServiceProvider.php
Created : /srv/projects_root/Modules/V2/Http/Controllers/V2Controller.php
Module [V2] created successfully.
Module [V2] used successfully.
```
This "magic" is done (behind the scene) by wonderful package laravel-modules, 
many thx to nWidart https://github.com/nWidart/laravel-modules 

And RAML-types based generated files:
```sh
================ Tag Entities
Modules/V3/Http/Controllers/DefaultController.php created
Modules/V3/Http/Controllers/TagController.php created
Modules/V3/Http/Middleware/TagMiddleware.php created
Modules/V3/Entities/Tag.php created
Modules/V3/Http/routes.php created
database/migrations/24_12_2016_1920702_create_table_tag.php created
================ Article Entities
Modules/V3/Http/Controllers/ArticleController.php created
Modules/V3/Http/Middleware/ArticleMiddleware.php created
Modules/V3/Entities/Article.php created
database/migrations/24_12_2016_1920646_create_table_article.php created
...
```

Migration option: if U want to automatically create migrations for those entities of RAML - set ```--migrations``` key, ex.: 
```sh
php artisan raml:generate raml/articles.raml --migrations
```

Generated migrations will look like standart migrations in Laravel:
```php
<?php
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateArticleTable extends Migration 
{
    public  function up() {
        Schema::create('article', function(Blueprint $table) {
            $table->increments('id');
            $table->string('title');
            $table->string('description');
            $table->string('url');
            // Show at the top of main page
            $table->unsignedTinyInteger('show_in_top');
            $table->timestamps();
        });
    }

    public  function down() {
        Schema::create('article', function(Blueprint $table) {
            $table->dropIfExists('article');
        });
    }


}
```
### RAML Types and Declarations

The ```version``` root property !required
```RAML
version: v1
```
converts to ```/Modules/V1/``` directory.

Types ``` ID, Type, DataObject/DataArray``` are special helper types - !required
```RAML
  ID:
    type: integer
    required: true
  Type:
    type: string
    required: true
    minLength: 1
    maxLength: 255
  DataObject:
    type: object
    required: true
  DataArray:
    type: array
    required: true
```

Special data type ``` RelationshipsDataItem ``` - !required
```RAML
  RelationshipsDataItem:
    type: object
    properties:
      id: ID
      type: Type
```
defined in every relationship custom type

Attributes ```*Attributes``` are defined for every custom Object ex.:
```RAML
  ArticleAttributes:
    description: Article attributes description
    type: object
    properties:
      title:
        required: true
        type: string
        minLength: 16
        maxLength: 256
      description:
        required: true
        type: string
        minLength: 32
        maxLength: 1024
      url:
        required: false
        type: string
        minLength: 16
        maxLength: 255
      show_in_top:
        description: Show at the top of main page
        required: false
        type: boolean
      status:
        description: The state of an article
        enum: ["draft", "published", "postponed", "archived"]
```

Relationships custom type definition semantics ```*Relationships```
```RAML
  TagsRelationships:
    description: Tag relationship description
    type: object
    properties:
      data:
        type: DataArray
        items:
          type: RelationshipsDataItem
```

Complete composite Object looks like this: 
```RAML
  Article:
    type: object
    properties:
      type: Type
      id: ID
      attributes: ArticleAttributes
      relationships: TagsRelationships
```
That is all that PHP-code generator needs to provide code structure that just works out-fo-the-box within Laravel framework, 
where may any business logic be applied

Complete directory structure after generator will end up it`s work will be like:
```php
Modules/{version}/Http/Controllers/ - contains controllers that extends the DefaultController (descendant of Laravel's Controller)
Modules/{version}/Http/Middleware/ - contains forms that extends the BaseFormRequest (descendant of Laravel's FormRequest) and validates input attributes (that were previously defined as *Attributes in RAML)
Modules/{version}/Entities/ - contains mappers that extends the BaseModel (descendant of Laravel's Model) and maps attributes to RDBMS
```
DefaultController example:
```php
<?php
namespace Modules\V1\Http\Controllers;

class ArticleController extends DefaultController 
{

}
```
By default every controller works with any of GET - index/view, POST - create, PATCH - update, DELETE - delete methods.
So You don't need to implement anything special here.

Validation BaseFormRequest example:
```php
<?php
namespace Modules\V1\Http\Middleware;

use rjapi\extension\BaseFormRequest;

class ArticleMiddleware extends BaseFormRequest 
{
    public $id = null;
    // Attributes
    public $title = null;
    public $description = null;
    public $url = null;
    public $show_in_top = null;
    public $status = null;

    public  function authorize(): bool {
        return true;
    }

    public  function rules(): array {
        return [
            "title" => "required|string|min:16|max:256",
            "description" => "required|string|min:32|max:1024",
            "url" => "string|min:16|max:255",
            // Show at the top of main page
            "show_in_top" => "boolean",
            // The state of an article
            "status" => "in:draft,published,postponed,archived",
        ];
    }

    public  function relations(): array {
        return [
            "tags",
        ];
    }
}
```

BaseModel example:
```php
<?php
namespace Modules\V1\Entities;

use rjapi\extension\BaseModel;

class Article extends BaseModel 
{
    protected $primaryKey = "id";
    protected $table = "article";
    public $timestamps = false;
}
```

### Turn off JSON API support
If you are willing to disable json api specification mappings into Laravel application (for instance - You need to generate MVC-structure into laravel-module and make Your own json schema, or any other output format), just set ```$jsonApi``` property in DefaultController to false:
```php
<?php
namespace Modules\V1\Http\Controllers;

use rjapi\extension\BaseController;

class DefaultController extends BaseController 
{
    protected $jsonApi = false;
}
```
As this class inherited by all Controllers - You don't have to add this property in every Controller class.
By default JSON API is turned on.

Laravel project example with generated files can be found here -  https://github.com/RJAPI/rjapi-laravel 

To get deep-into ```RAML``` specification - https://github.com/raml-org/raml-spec/blob/master/versions/raml-10/raml-10.md/

To get deep-into ```JSON-API``` specification - http://jsonapi.org/format/
JSON-API support is provided, particularly for output, by Fractal package - http://fractal.thephpleague.com/

Happy coding ;-)