# How to create multilingual sites in Silverstripe CMS

Part 1 - Introduction to i18n

In Europe, we have many countries with different languages spoken. If you want to sell your products or services to a broader audience, you should think about making your website or webapp multilingual. Pretty early in my career as software developer I had to deal with this topic. Back then I tried to abstract multilingual content and keep all the surrounding HTML the same. Some of those pretty old websites still work today, though they're pretty outdated. Then I learned about Silverstripe CMS and its i18n features. But... what does i18n actually mean?


## What does i18n mean?
I18n is the abbreviation of "internationalisation". That word has 20 characters in total, so we take the first and last character and the remaining 18 characters "nternationalisatio" is shortened to 18. Voilà! i18n is born. Developers sometimes have a strange sense of humour, don't they?

In practical terms, it means that you can design software that can be adapted to various languages and regions without changing the code.

Another related abbreviation is L10n, which stands for "localisation" - which is the process of adding new languages to the software.

## Language versus Locale:

While language and locale are often used interchangeably, they are not the same. 
A language can be spoken in many countries or regions. For the German language there are at least three locales. For Germany, Austria and Switzerland the locales are `de_DE`, `de_AT` and `de_CH`. One famous difference between `de_DE` and `de_AT` is e.g. the name of the first month of the year, Januar in Germany and Jänner in Austria.

Tip: If your website is in English, but for international guests, (not specifically for British or American people), there is even a locale for "international English" - en_100.

You can find more about [i18n and l10n on wikipedia](https://en.wikipedia.org/wiki/Internationalization_and_localization).

## What is affected by i18n?

The most obvious part that's different in the various locales is content. That can be text in the main content but also text on buttons, form labels, even invisible text in metadata. You also have to think about using the right character set, which, nowadays, is commonly utf-8. Some languages like Arabic are written right-to-left, so you might have to keep this in mind during your build.

For text in your UI there might be different pluralisation rules beyond just simply adding a "s" to the word like in English. But also different formats for:

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

First, we need to modify our software so that it can translate content. That's basically i18n. There are different ways to achieve this, but generally we end up having a function like `translateText($key, $locale)`, which translates the desired text into the given locale. The way the translations are actually stored can be determined on a per-application basis.

With i18n functionality in place, we can add translated content during the process of L10n, localisation. Here we translate the required text into different locales, depending on our needs.

The third step is called quality assurance, ensuring that the translations are easy to understand and culturally appropriate for the given audience.

How can Silverstripe CMS help you with i18n? That's what we will learn in the next parts of this series.
