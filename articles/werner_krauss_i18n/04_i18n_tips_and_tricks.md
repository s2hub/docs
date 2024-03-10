# Tips, Tricks, Tracs

Locale vs Countries

Most of the time you can abuse the locale switcher as a country switcher, but sometimes a country has more than one official languge. E.g. Switzerland has German, French and Italian, Finland has Finnish and Swedish, New Zealand has English and Maori...

And on the other hand - a language can be spoken in many countries. I18N can sometimes be pretty tricky, isn't it?

Partial caching and different locales

If you use partial caching, $CurrentLocale should be part of the cache key. Either globally configured or in each cache key seperately. Been ther, done that. It's annoying, when only one locale is cached - and displayed in other locales and doesn't match your user's locale.

## TODO: db fields syntactic sugar (see 02_i18n_and_silverstripe_translating_static_content.md); use in field labels and relation labels

## Useful Extensions and Ideas

derralf/silverstripe-fluent-tweaks
set order / sorting
hide locale from Menu
hide from MetaTags
level51/silverstripe-fluent-autotranslate
automatic translation via Google Translate API
arrilo/silverstripe-deepl-translator
deepl translations for Silverstripe CMS
Translating via deepl/ChatGPT