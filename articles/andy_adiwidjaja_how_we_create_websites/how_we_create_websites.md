# How we create websites with Silverstripe CMS

## Introduction
TBD

The idea of our setup is:
- we nearly never use themes in our projects
- we like to start frontend from scratch
- separate frontend from backend
- use modern javascript and scss
- seldom use translations
- use as few modules as possible

## Preconditions
We are working on standard apache environments, meaning we have an apache pointing to a www directory in our home directory where we have the projects as subfolders. We can switch php versions. Please have a look at the official server requirements when in doubt: 

https://docs.silverstripe.org/en/5/getting_started/server_requirements/

We had some issues with discontinued module in the past (going from 2 to 3, but also in other systems), so we try to use as few modules as possible. 
## Start
So, we start off as in the documentation:

```bash
composer create-project silverstripe/installer example-website
cd example-website
git init
git remote add origin git@github.com:atwx/silverstripe-elemental-vite-template.git
```

The standard install includes the simple-theme. We don't need that, instead we install elemental and a vitehelper module. We are also adding ideannotator.

```bash
composer remove silverstripe-themes/simple
composer require dnadesign/silverstripe-elemental
composer require atwx/silverstripe-vitehelper
composer require --dev silverleague/ideannotator
```

We need a `.env` config file in the main directory:

```
SS_ENVIRONMENT_TYPE="dev"  
SS_DATABASE_SERVER="127.0.0.1"  
SS_DATABASE_NAME="vitetemplate"  
SS_DATABASE_USERNAME="root"  
SS_DATABASE_PASSWORD="root"  
SS_DEFAULT_ADMIN_USERNAME="admin"  
SS_DEFAULT_ADMIN_PASSWORD="password"

# VITE_DEV_SERVER_URL="http://localhost:3000"
```

I will explain `VITE_DEV_SERVER_URL` later. Before building, we add a config file for ideannotator:

`/app/config/ideannotator.yml`

```yml
---  
Only:  
  environment: 'dev'  
---  
SilverLeagueIDEAnnotatorDataObjectAnnotator:  
  enabled: true  
  enabled_modules:  
    - app
```

To cite the description of ideannotator: "This module generates @property, @method and @mixin tags for DataObjects, PageControllers and (Data)Extensions, so ide's like PHPStorm recognize the database and relations that are set in the $db, $has_one, $has_many and $many_many arrays."

This is very useful to work with Silverstripe CMS. Now is the time to build the database.

```bash
vendor/bin/sake dev/build
```

If everything works, you should see the output of building the database. Before the initial commit, I will add some entries to the `.gitignore`:

```.gitignore
/silverstripe-cache/  
/.env  
/vendor/  
/_resources/  
/public/_resources/  
/.graphql-generated/  
/public/_graphql/  

# Added
/.idea/  
/app/client/dist/
/node_modules
```

The `.idea` is the config directory of my IDE (PHPStorm), `/app/client/dist/` will be the build folder of our frontend tools. We are always building the frontend in the deployment so we don't want it in the repository.

Now we can commit:

```
git add .
git commit -m "Initial commit"
git push --set-upstream origin main
```
## Elemental configuration
We are always using elemental for websites. Elemental needs a configuration on which page it is enabled. In addition, we remove a standard element.

`app/_config/elemental.yml`

```yml
---  
Name: app-elemental  
---  
Page:  
  extensions:  
    - DNADesignElementalExtensionsElementalPageExtension  
  disallowed_elements:  
    - DNADesignElementalModelsElementContent
```

This configuration will add the `ElementalArea` to the standard `Page` and shows the elemental editor in the CMS. In most projects, we can just add `ElementalPageExtension` to the `Page`. If you need some pages without `ElementalArea` you will want create a subclass `ElementalPage` so that `Page` stays without `ElementalArea`.

Build again!

```bash
vendor/bin/sake dev/build
```

You can now open the CMS in the browser and you will see empty elemental areas.
## Templates
You can now open the page in the browser, but it will be very basic, because it uses the default template `SilverStripe/Control/Controller.ss` if no other template is present.

We nearly never use themes, so we put the templates in `app/templates`. The basic structure is a `app/templates/Page.ss` for the main template and a `app/templates/Layout/Page.ss` for the page type:

`/app/templates/Page.ss`:
```html
<!doctype html>  
<html lang="de">  
    <head>        
	    <% base_tag %>  
        $MetaTags(false)  
        <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">  
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />  
        <meta charset="utf-8">  
        <title>$Title - $SiteConfig.Title</title>  
    </head>
    <body>
		<% include Header %>  
        <main class="main">  
                $Layout  
        </main>  
    </body>
</html>
```

`/app/templates/Layout/Page.ss`:

```html
$ElementalArea  
<% if $Form %>  
    $Form  
<% end_if %>
```

`/app/templates/Includes/Header`:

```html
<header class="header">
    <nav class="main-nav">  
        <ul class="nav">  
            <% loop $Menu(1) %>  
                <li class="nav_link<% if $LinkOrSection == "section" %> nav_link--active<% end_if %>">  
                    <a href="$Link">$MenuTitle</a>  
                </li>            
            <% end_loop %>  
        </ul>  
    </nav>
</header>
```
## Frontend building
We are always using SCSS and we are using various technologies in JavaScript depending on the project - most often vanilla javascript, sometimes React and sometimes ViteJS. If the project is very frontend-heavy, we are using a headless setup with NextJS, but that is out of scope for this tutorial.

Just recently we switched from `Webpack` to `ViteJS` for frontend building. `ViteJS` is incredibly during development, but the configuration (like in any builder) sometimes needs some time. We are using `yarn`, but that is just a matter of taste. We are nearly never using css frameworks (we are uising `uikit` for some tools), just `normalize.css` and `include-media`, a helper for easier media queries.

We also add some scripts so your `package.json`  should look like this:

```json
{  
  "name": "silverstripe-vite-template",  
  "version": "1.0.0",  
  "private": false,
  "type": "module",  
  "scripts": {  
    "dev": "vite",  
    "build": "vite build"
  },  
  "devDependencies": {  
    "sass": "^1.71.1",  
    "vite": "^5.1.4"  
  }  
}
```

So, here is the simplest vite configuration.

`/vite.config.js`

```js
import {defineConfig} from 'vite'  

// https://vitejs.dev/config/  
export default defineConfig(({command}) => {  
  return {  
    server: {  
      host: '0.0.0.0',  
      port: 3000,  
    },  
    base: './',  
    publicDir: 'app/client/public',  
    build: {  
      outDir: './app/client/dist',  
      manifest: true,  
      sourcemap: true,  
      rollupOptions: {  
        input: {  
          'main.js': './app/client/src/js/main.js',  
          'main.scss': './app/client/src/scss/main.scss',  
          'editor.scss': './app/client/src/scss/editor.scss',
        }  
      },  
    },  
  }  
})
```

The thing that took me a whole day to figure out was `base: './',`. The default is `/` and this leads to absolute asset paths which don't work on the build.

So we are starting with a basic `/app/client/src/js/main.js`:

```js
// Intentionally blank
```

In the scss folder, we add something more:

`app/client/src/scss/variables.scss`:

```scss
$maxWidth: 1200px;  
$maxWidthContent: 980px;  
  
$breakpoints: (  
  'smartphone': 480px,  
  'medium': 700px,  
  'desktop': 1000px,  
  'maxWidth': $maxWidth,  
);  
  
$colorPrimary: #8C3337;  
$colorSecondary: #C9A141;
```

Just some common variables and colours. The breakpoints are definitions for the `include-media` module. Sometimes, it's incredible useful to change breakpoints site-wide by changing a variable.

`app/client/src/scss/typography.scss`:

```scss
body {  
    font-family: 'Roboto', sans-serif;  
    font-size: 16px;  
    font-weight: 400;  
}
```

We have a separate typography.scss so that we can include it in the editor.scss:

`app/client/src/scss/editor.scss`:

```scss
@import "include-media";
// TBD: @import "fonts";  
@import "normalize.css";  
  
@import "variables";  
@import "typography";
```

`app/client/src/scss/main.scss`:

```scss
@import "include-media";  
@import "normalize.css";  
  
//@import "fonts";  
@import "variables.scss";  
@import "typography.scss";  
  
@import "layout/header.scss";  
//@import "elements/TextImageElement.scss";
```

`/app/client/src/scss/layout/header.scss`

```scss
.header {  
    background-color: $color-primary;  
    padding: 10px 20px;  
  
    .main-nav {  
        .nav {  
            list-style: none;  
            margin: 0;  
            padding: 0;  
            display: flex;  
            .nav_link {  
                a {  
                    color: $color-white;  
                    text-decoration: none;  
                    padding: 10px 20px;  
                    display: block;  
                }  
  
                &.nav_link--active {  
                    a {  
                        background-color: $color-white;  
                        color: $color-primary;  
                    }  
                }  
            }  
        }  
    }  
}
```

Now you can run a first build:

```bash
yarn build
```

This should create the dist directory with the compiled `js` and `css` files and the `.vite`-Directory with the manifest.

To use the compiled files in the templates, we need to expose them. For that, we add the directory `app/client/dist` to the `extra/expose` configuration in `composer.json`

```json
{  
	...
    "extra": {  
        "expose": [  
            "app/client/dist"  
        ],  
        ...
    }
}
```

Often, we also have additional folders with static assets, which we expose here, like `app/client/images`. Assets that are referenced from `scss` like fonts don't have to be exposed separately, they will be copied into `app/client/dist`.

Remember to run `vendor-expose`. This will create the soft links in the public directory.

```bash
composer vendor-expose
```

Now we can use the compiled files in the templates. For this, we can use helper functions from `atwx/silverstripe-vitehelper`:

```app/template/Page.ss```:

```html
<!doctype html>  
<html lang="de">  
    <head>        
	    <% base_tag %>  
        $MetaTags(false)  
        <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">  
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />  
        <meta charset="utf-8">  
        <title>$Title - $SiteConfig.Title</title>  
        $ViteClient.RAW
        <link rel="stylesheet" href="$Vite("app/client/src/scss/main.scss")">  
    </head>
    <body>
		<% include Header %>  
        <main class="main">  
                $Layout  
        </main>  
        <script type="module" src="$Vite("app/client/src/js/main.js")"></script> 
    </body>
</html>
```

The `$Vite()` function provides a helper which extracts the correct path in the `manifest.json`. You can configure the `output_url` and the `manifest_path` if you are using different folders.

The `$ViteClient` is the vite development client. To use it, we have to configure a vite development server. We already did that in the `vite.config.js`. We just need to uncomment the variable `VITE_DEV_SERVER_URL` in the `.env`, then we can start the development server:

```bash
yarn dev
```

Then, every change in the `js` and `scss` files will be updated immediately. It's likely that the browser reloads faster than you can switch to the browser!
## Editor CSS
In Silverstripe CMS, we can customise the styling of content editors in the CMS. This is important to give editors a hint how they content will look on the website. Normally we would configure the `editor_css` in the `TinyMCEConfig`, but the build process always add a hash to the filename. Instead, the `ViteHelper` class provides a configuration property:

```yml
AtwxViteHelperHelperViteHelper:  
  editor_css: 'app/client/src/scss/editor.scss'
```

This looks up the correct filename in the `manifest.json`.

## Creating elemental blocks
As a first block, we create a standard text-image-block. For this we basically create three files:
- PHP: `/app/src/Elements/TextImageElement`
- Template: `/app/templates/App/Elements/TextImageElement.ss`
- SCSS: `app/client/src/scss/elements/TextImageElement.scss`

`/app/src/Elements/TextImageElement`:

```php
<?php  
  
namespace AppElements;  
  
use DNADesignElementalModelsBaseElement;  
use SilverStripeAssetsImage;  
use SilverStripeFormsDropdownField;  
  
/**  
 * Class AppElementsTextImageElement * * @property string $Text  
 * @property string $Variant  
 * @property int $ImageID  
 * @method SilverStripeAssetsImage Image()  
 */
class TextImageElement extends BaseElement  
{  
    private static $db = [  
        "Text" => "HTMLText",  
        "Variant" => "Varchar(20)",  
    ];  
  
    private static $has_one = [  
        "Image" => Image::class,  
    ];  
  
    private static $owns = [  
        "Image"  
    ];  
  
    private static $field_labels = [  
        "Text" => "Text",  
        "Image" => "Bild",  
    ];  
  
    private static $table_name = 'TextImageElement';  
    private static $icon = 'font-icon-block-promo-3';  
  
    public function getType()  
    {  
        return "Text + Bild";  
    }  
  
    public function getCMSFields()  
    {  
        $fields = parent::getCMSFields();  
        $fields->replaceField('Variant', new DropdownField('Variant', 'Variante', [  
            "left" => "Bild links",  
            "right" => "Bild rechts",  
        ]));  
        return $fields;  
    }  
}
```

`/app/templates/App/Elements/TextImageElement.ss`:
```html
<div class="section section--text-image section--text-image-$Variant">  
    <div class="section_content">  
        <div class="section_content_image">  
            $Image.ScaleMaxWidth(600)  
        </div>  
        <div class="section_content_text">  
            <% if $ShowTitle %><h2>$Title</h2><% end_if %>  
            <p>$Text</p>  
        </div>
	</div>
</div>
```
 
 `/app/client/src/scss/elements/TextImageElement.scss`:
```scss
.section--text-image {  
  
    .section_content {  
        .section_content_image {  
            width: 100%;  
  
            img {  
                display: block;  
                width: 100%;  
            }  
        }  
    }  
  
    @include media(">desktop") {  
        .section_content {  
            display: flex;  
  
            .section_content_image, .section_content_text {  
                width: 50%;  
            }  
  
            .section_content_text {  
                padding-left: 20px;  
            }  
        }  
  
        &.section--text-image-right {  
            .section_content {  
                .section_content_image {  
                    order: 2;  
                }  
  
                .section_content_text {  
                    order: 1;  
                    padding-left: 0;  
                    padding-right: 20px;  
                }  
            }  
        }  
    }  
}
```

## Conclusion
TBD