# I18n and Silverstripe - Translating dynamic content
## How to translate dynamic content in Silverstripe CMS

Isn't a CMS a Content Management System, where you can add content of your website in a nice and user-friendly editor in your browser? Yes, of course. After learning the basics of how to translate static text in your website, the next question is how to translate all the static content that is saved in the database.

For this we use module called Fluent. You can install it with composer and after running dev/build you can configure your desired locales in the CMS.

All pages are translated automatically, cause the module adds its extension to SiteTree. If you have other DataObjects you need to be translated into various languages, you can add the FluentExtension to them and define, which fields should be translated and which not.

TODO: code example

You should add Fluent pretty much at the beginning of your project. Adding it later is a bit tricky, because you need to publish the pages again.

Fluent also has nice features for bigger websites. You can either configure a domain per locale and/or distinguish the current locale by a URL segment. www.foo.com/pt/empregados is e.g. the Portuguese version of www.foo.com/en/team.

If you have many locales of a given language (e.g. sometimes different content for Germany, Austria and Switzerland), you can define fallback locales. This way you only need to translate the DataObjects that have different content, the other ones are the same in all locales. This can be handy if e.g. product data is the same in all locales, but some legal texts or imprint need different content for some reason.

TODO: Screenshot Fluent configuration

Tip: If you want the locale configuration in your git repository instead, you can use Silverstripe's default_records functionality.

TODO: code example

## How to translate the content actually

When fluent is installed and configured, you're ready to translate your site's content in the CMS. Each page can be translated in all configured locales. In the locales tab you see which locales are already translated or not and you also can copy whole DataObjects from or to other locales.
You also have translation related actions near your "save" and "publish" buttions, e.g. "save and publish all locales".

## Translating DataObjects

For translating DataObjects you need to add FluentExtension. If you do nothing else, all text fields are translated automatically. Of course you can configure the fields you need translated. Let's say, you have a DataObject for Resellers of your products, you have fields like "Title", "Street", "Town", etc. Then you can translate e.g. only "Title" and "Town", cause the reseller could have a sligtly different name in other languages, and town names can also be different. The Bavarian capital city München is written Munich in English and Monaco di Baviera in Italian.

Translating Custom Form Fields in the CMS

If you have a subclass of Page, which has FluentExtension applied, you notice, that your own fields won't have that nice badge from fluent and will not be translated. That's because, FluentExtension adds this badges using the updateCMSFields() hook while still on Page and before the FieldList is returned to your subclass. The solution to this problem is to define your fields inside $this->beforeUpdateCMSFields(). Then the fields are added before it's Fluent's turn to add its functionality.

Translating Relations

Sometimes you want different objects related in has_many or many_many relations. Fluent has the FluentFilteredExtension to help you with this problem. This extension filters the relation in the frontend and in the backend you can choose, which item should be available in which locale.
How to switch locales in the frontend

Fluent also has a built in helper to create a locale switcher. It basically loops over all configured locales and returns a link to the current page in another locale, or to home, if the current page is not translated in that specific locale. If you need, e.g. a flag for each locale, you can add this via an Extension and show them in your locale switcher.
How to deal with localised content in your code.

Sometimes you need to manipulate content in another locale than the current one. E.g. for tasks that translate your content and should save it to another locale or if you want to read something in another locale, e.g. for exporting content in different locales at once. Then you need FluentState, your friend when dealing with localised content programatically.

This class has all useful information about the current settings: current locale and domain, if the site is in domain mode or not and if you're in the frontend or not.

FluentState::singleton()->withState() changes the locale for a specific code inside a closure.

TODO: code example

