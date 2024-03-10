# How to create multilingual sites in Silverstripe CMS

Part 1 - Introduction to i18n

In Europe, we have many countries with different languages spoken. If you want to sell your products or services to a broader audience, you should think about making your website or webapp multilingual. Pretty early in my career as software developer I had to deal with this topic. Back then I tried to abstract multilingual content and keep all the surrounding HTML the same. Some of those pretty old websites still work today, though they're pretty outdated. Then I learned about Silverstripe CMS and its i18n features. But... what does i18n actually mean?


## What does i18n mean?
I18n is the abbreviation of "internationalisation". That word has 20 characters in total, so we take the first and last character and the left 18 characters "nternationalisatio" is shortened to 18. Voila, i18n is born. Developers sometimes have a strange sense of humour, don't they?

It actually means that you can design a software, that can be adapted to various languages and regions without changing the code.

Another abbreviation is L10n, which stands for "localisation". That's the process to add new languages to the software

## Language versus Locale:

While language and locale are often used interchangeably, they are not the same. 
A language can be spoken in many countries or regions. For German language there are at least three locales. For Germany, Austria and Switzerland the locales are `de_DE`, `de_AT` and `de_CH`. One famous difference between `de_DE` and `de_AT` is e.g. the name of the first month of the year, Januar in Germany and Jänner in Austria.

Tip: If your website is in English, but for international guests, not especially for British or American people, there is even a locale for "international Englishj", en_100.
You can find more about [i18n and l10n on wikipedia](https://en.wikipedia.org/wiki/Internationalization_and_localization).

## What parts are affected by i18n?

The most obvious part that's different in various locales is content. That can be text, the main content but also text on buttons, form labels, even invisible text in metadata. You also have to think about the right character set, which is nowadays most of the time utf-8. Some languages like Arabic are written right-to-left, so you might have to keep this in mind.

For texts in your UI there might be different pluralisation rules, not just simply adding a "s" to the word like in English. But also different formats for:

* date and time
* calendar systems
* time zones
* number and currency formats
* quotes... 

There can be many differences even between neighbour countries.

And of course, you should add the current locale in your `<html lang="...">` tag.

You see, there can be many things to think about when you're planning to create a website or webapp that should work for an international market.

## How to make software multilingual
The process of making software multilingual is pretty standardised.

First, we need to modify our software that it can translate content. That's basically  i18n. There are different ways to make this, but everytime we end up having a function like `translateText($key, $locale)`, which translates the desired text into the given locale. Then it doesn't matter for us where the translations are actually stored.

Next we can add translated content during the process of L10n, localisation. Here we translate the text that can be translated into different locales, depending on our needs.

The third step is called quality assurance. When you have e.g. international tourists, the English translation should be easy to understand und doesn't need to comped with Shakespeare, cause your visitors might not speak the language that well.   You might also check, if translations are culturally appropriate for the given audience.

How can Silverstripe CMS help you with i18n? That's what we will learn in the next parts of this series.
