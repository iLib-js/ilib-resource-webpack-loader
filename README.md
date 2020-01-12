# ilib-resource-webpack-loader

ilib-resource-webpack-loader is a webpack loader for translated resource files that
allows your application to include those files automatically within the webpacked
application.

# Using the Resource Loader

## webpack.config.js

To add translated resources to a webpacked application, modify the
rules section of the application's webpack.config.js file to add
configuration for the loader:

```
    module: {
        rules: [{
            test: /.js$/,
            use: {
                loader: 'ilib-resource-webpack-loader',
                options: {
                    locales: ["en-US", "de-DE", "ja-JP" ],
                    mode: 'development',
                    assembly: 'assembled',
                    resourceDirs: [
                        path.resolve('./assets/translations'),
                        path.resolve('./src/components/res')
                    ]
                }
            }
        }]
    },
```

Here are what the options mean:

- locales - an array of BCP-47 style tags that specify the list of locales that
  the application needs to support. (See below for more details.)
- mode - 'development' vs. 'production'
    - in 'development', files are included with formatting, not compressed
    - in 'production' mode, files will be compressed/uglified
- assembly - "assembled" vs. "dynamic"
    - in "assembled" mode, the translations are included directly into
      the webpack bundle. In "dynamic" mode, translated files are put
      into separate webpack chunks that are loaded dynamically when needed.
- resourceDirs - an array of paths where resource files can be found
    - an array allows for including resource files from
      multiple parts of an applications which can be released
      and localized separately
    - paths may be absolute or relative. When relative, they
      should be relative to the root of the applications.

## project.json

In the project.json configuration file for loctool, choose the "custom"
style of project and use the loctool plugins that make sense for the
application.

```
{
    "name": "My Web App",
    "id": "mywebapp",
    "projectType": "custom",
    "sourceLocale": "en-US",
    "resourceDirs": {
        "jst": "./i18n",
        "javascript": "./i18n"
    },
    "plugins": [
        "html",
        "javascript",
        "jst"
    ],
    "excludes": [
        "./.git",
        "./assets",
        "./bin",
        "./libs",
        "./script/**/*.sh"
    ],
    "includes: [
        "**/*.html",
        "**/*.js",
        "**/*.jst"
    ],
    settings: {
        locales: ["es-ES", "de-DE", "fr-FR"],
    }
}
```

In the `plugins` property, list all of the loctool plugins necessary
for all of the file types in the project, and make sure the `resourceDirs`
property contains the output resource directory for each file type. These
are the directories where the loctool will write the resource files. This
should match the `resourceDirs` property in the webpack.config.js file
so that the resource loader can find those resource files and include them.

## Configuration Choices

To configure the loader, you will need to decide upon:

- Which locales does the application need?
- Do you want to assemble the translation
  data directly into the the app's webpack
  bundle, or should they be dynamically lazy-loaded?
- Is the application being built for debug or production mode?

### Which Locales?

The loader is configured by default to support the top 20 locales around the world in
terms of Internet traffic. If no locales are explicitly chosen, the loader will default
to the translations for these top 20 locales if they exist. That is a very small subset
of all the locales that could be supported, yet the set of translated strings could still
be potentially large.

When the loader is including translations, it first checks if any of the translated
resource files for the given locales are available in any of the resources directories.
If they are not there, it does not need to include them, of course, and it silently
skips those locales.

If the app does not support that many locales, it can have a significantly
smaller footprint by specifying a smaller set of locales in the webpack.config.js.

Locales should be specified using [BCP-47 locale tags](https://en.wikipedia.org/wiki/IETF_language_tag)
(aka. IETF tags). This uses ISO 639 codes for languages, ISO 15924 codes for scripts,
and ISO 3166 codes for regions, separated by dashes. eg. US English is "en-US" and
Chinese for China written with the simplified script is "zh-Hans-CN".


### Which Files are Included?

The loader automatically includes all files that use the locale spec somewhere in the
file name. The file name may have other characters in it as well. 
For example, if the locale is "de-DE", then all of the files "resources-de-DE.json",
"strings_de-DE.json" and "de-DE-text.json" are all included.

Ilib's ResBundle class has a concept of "locale fallbacks", which allows the application
to use the translated resources of other locales if the strings are not available
in the current locale. Right now, the fallbacks are simply variations on the full
locale, but in the future may include other locales. Accordingly, this resource
loader will include all of the files for the fallback locales as well.

The fallback locale variations are as follows:

- language-script-region-variant
- language-region-variant
- language-script-region
- region-variant
- language-region
- language-script
- region
- language
- root

Each of "language", "script", "region" and "variant" are replaced with the corresponding
part from the full locale spec if they are available.

"Root" is a special locale that all locales derive from, and usually contains text
written in the source language (typically English). The root in ilib also contains
any data that is not locale-sensitive.

So, for example, if the full locale spec is "zh-Hans-CN", then the list of variations that
the loader will look for are:

- zh-Hans-CN
- zh-CN
- zh-Hans
- CN
- zh
- root

Note that in this example, there is no "variant" part in the full locale spec, so any
fallback locale that uses the variant is simply left out.

The list of fallbacks can be a little tricky -- if full locale spec is 
"de-DE" for example, then any file name that includes the language spec "de" will
be included, including such file names as "device.json" or "delimiter.json" because
they happen to include the substring "de". Those are probably not locale files. As such,
the developer should be careful which files appear in the resource directories. Only
localized files should appear there. Unless you need to have multiple files for the
same locale, it is best just to name the file simply for the locale, such as "de-DE.json"
or "de.json". 

### Assembled or Dynamic Translations?

There are two ways to include the translation data into the webpack configuration for an
application: assembled and dynamic.

1. Assembled. Translations are included directly into the app's webpack bundle.

   Advantages:
   - everything is loaded and cached at once
   - all translations are available for synchronous use as soon as the browser has
     loaded the webpacked js file. No async calls, callbacks, or promises are needed!
   - there are less files to move around, push to production, or to check into your repo

   Disadvantages:
   - that single file can get large quickly if there are a lot of locales or a lot of strings
   - the app would load all locales at once, even if you only use one locale at a time,
     means extra network bandwidth and load time for data, as well as memory and
     disk footprint that the user isn't actually using

   Assembled translations are a good choice if the app only supports a few locales or
   if there are only a few strings to translate. The first
   time a user hits the website, they download a larger-than-needed webpack file, but then
   it is cached and everything after that is relatively simple.

2. Dynamic. Webpack has the ability to lazy-load translations as they are needed. With
   this type of configuration, the code is assembled into one file, but the translation
   data goes into other separate webpack chunk files, which are lazy-loaded as they
   are needed.

   Advantages:
   - file size and therefore initial page load time are minimized
   - the application code can be cached in the browser and doesn't change often even if
     the translations do
   - the translation files can be cached separately, allowing the app to add new locales
     to the web site later without affecting any existing cache for the translations in
     other locales

   Disadvantages:
   - the number of chunk files can get unwieldy if the app has a lot of locales. The
     loader will create a different chunk file for every locale variation it finds.
   - since webpack loads the bundles asynchronously, the app must use the ilib ResBundle class
     asynchronously with callbacks or promises. Alternately, the app must pre-initialize the
     translation data asynchronously and then wait for the load to finish in a jquery
     "document ready" kind of callback function before using the translations synchronously
     after that.

   Using dynamic data is a good choice if you have a lot of locales or have a lot of strings
   to translate.

### Development vs. Production mode

The mode property can be set to one of the modes "development" or "production".
In development mode, a developer typically wants to see nicely formatted code and data to
make debugging easier. Load time and footprint do not matter so much. In this mode, the
translated output is generated with formatting and line breaks. In production mode,
load time and footprint are much more of a concern, so translation files are
compressed/uglified as they are added to the webpack bundle.
 

## How Does the Loader Work?

Webpack loaders in general are used to read and modify javascript files as they are
added to the webpack bundle. The ilib-webpack-loader does this as well. Specifically,
it is doing these things:

* If the javascript file being added to the bundle happens to be the iLib ResBundle
class, it modifies the code to depend on a hidden class that knows how to load the
translated files.
* It also writes out the code for the hidden file. This code loads the translated
files depending on a locale parameter. This code is different depending on whether
the configuration specifies assembled or dynamic translations, and whether it specifies
development or production mode.   

# The Translation Cycle for an Application

Applications should be translated frequently so that stories in a sprint can be
considered "done" properly. Testers will need the translations to verify that the
feature is "done".

## Translation of a Regular Webpacked Application

To get a React application translated, see the next section.
To get a non-React Javascript application translated, follow these steps:

1. Make sure to add a dependency on ilib and a devDependency on
  ilib-resource-webpack-loader to the package.json file
1. Import and instantiate an ilib `ResBundle` instance in each file
  of the application classes that contains translatable strings specifying
  the path where the translated resource files will go or create a
  `ResBundle` instance once for the whole application and pass it
  around to the rest of the code
1. Make sure all user-visible strings in the application are wrapped
  in a `ResBundle.getString()` or `ResBundle.getStringJS()` call
1. Make sure the application includes a way for a user to change their
  locale, either by adding a choice in the UI or by having an admin
  change it for them.
    - It is recommended that every user have their own locale
      which is stored along with the user's information in the
      backend
1. Add loctool to the devDependencies in the package.json file and make
  sure it is installed and that node_modules/.bin is in the path (or you
  can use npx to run it)
    - Follow the directions in the loctool README to create
      a loctool project.json file for the application and commit it
      to your repo
1. Run the loctool to extract all of the localizable strings into xliff
  files
1. Send the xliff files out for translation
1. When the translated xliff files are returned, use the loctool again
  to generate the translated json resource files in the path specified
  in the second step above.
1. Add ilib-resource-webpack-loader in the webpack.config.js file with the
  appropriate options (according to the details above)
1. Run webpack as normal, and it should find and include all of the
  translated files that the loctool had just generated

The translations should now be available so that when the user changes
their locale, the translated strings should appear in the UI.

In step 6 above, the loctool will simulataneously generate an xliff file
containing the new strings that the developers checked in since the last
set of xliff files was generated and sent out for translation. This means
that you can send those new xliff files out as the next translation batch,
restarting the cycle over again. It is recommended that the application
is translated on a regular basis. In some projects that have "continuous
localization" as part of their process, the new strings are sent out
as frequently as multiple times per week, ensuring that the translators
get a constant stream of strings to translate, rather than a huge
batch at the end of the project. Waiting to the end is usually a bad idea
because it creates a schedule bottle neck.

## Localization Process for React Applications

The process for localizing React applications is essentially the same
as above with the following tweaks:

- In step 1 above, instead of adding dependencies on ilib directly, add
  dependencies on ilib-es6 and react-ilib. Ilib-es6 contains ES6 wrappers
  around the regular ilib classes which can be
  included in ES6 code normally using regular `import` statements.
  Ilib code is written in ES5 and must be
  included using `require()`. Also, ilib-es6 uses promises to support
  asychronous ilib calls instead of the callback functions that
  regular ilib uses. React-ilib is a library of formatters and
  utility components that make it easier to localize React apps.
- Make sure the top level app is wrapped in a `LocaleDataProvider`
  component from react-ilib, which loads the ilib locale data and the
  translated resources that this webpack plugin includes into the
  webpack bundle.
    - The `LocaleDataProvider` component loads the data
      asynchronously and then renders the rest of the app only
      when the data is finished being loaded. In this way, the
      app does not appear in
      English first and then re-rendered in another language
      which creates a very odd user experience.

Example index.js file:

```javascript
// put this in your main index.js file
import React from 'react';
import ReactDOM from 'react-dom';
import path from 'path';
import App from './App'; // this is your main App code

import { LocaleDataProvider } from 'react-ilib';

ReactDOM.render(
    <LocaleDataProvider
        locale="de-DE"
        translationsDir={[path.join(__dirname, "res")]}
        bundleName="resources"
        app={App}  // don't render the App until after the locale data is loaded!
    />,
    document.getElementById('root')
);
```

- For any user-visible JSX strings, wrap them in a `Translate` component,
  and use the `rb.getString()` and `rb.getStringJS()` methods for strings in
  the JS code
    - The loctool has plugins that know how to extract strings from jsx
      and JavaScript code
- For any component that needs translated strings, use the
  `injectRb()` function to make sure that the translations are available
  in the code. The `rb` variable is the resource bundle instance that the
  `LocaleDataProvider` component creates for you when it has finished loading
  the translated strings.

Example:

```javascript
import { Translate, injectRb } from 'react-ilib';

function App(props) {
    const { rb } = props;
    const string1 = rb.getString("Hello World 1!");
    return (
        <span>
            {string1}
        </span>
        <span>
            <Translate>Hello World 2!</Translate>
        </span>
    );
}

export default injectRb(App);
```

# Handling the Translated Files

Some file types are localized by extracting strings into a resource
file, which is the way that regular JavaScript
code works. This webpack loader helps to include those resource files
into your webpacked application. The translated resource files that 
the loctool produces can be checked in to your source control system.

Other file types are localized "by copy". That is, they are localized by
creating translated copies of the source files
with the text in them replaced with translations. Often, these
are more structured file types such as front end templates or XML.

For example, regular HTML files are often localized by copy.
Now if an engineer changes a non-localizable part of an HTML source file,
such as JavaScript code or CSS or HTML attributes, there is no
need for a new translation. Yet, the translated HTML files will be out
of sync with the original source files because the code does not
match.

The solution is to run the loctool with every build. Doing this will guarantee
that the non-localizable bits of the translated files will be the
same as the in the source file, even when no localizable strings
are changed.

The rule of thumb is this: if the application includes any file types
that are localized by copy, it is better to commit the xliff files to the
source control system
and generate the translated files with every build instead of committing
the translated resource files. If the application does not have any
file types that are localized by copy, then commit the translated
resource files.

File types that are typically localized by
copy include the following: HTML, XHTML, JSON, and any front-end templates
such as JST, HAML, or JSP. Check with the documentation of the loctool plugin
for the particular file type to see how it is localized.

