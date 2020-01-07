# ilib-resource-webpack-loader

ilib-resource-webpack-loader is a webpack loader for translated resource files that
allows your application to include those files within the webpacked application.


# The Translation Cycle for an Application

To get a Javascript application translated, follow these steps:

1. Make sure to add a dependency on ilib and a devDependency on
  ilib-resource-webpack-loader to the package.json file
1. Import and instantiate an ilib `ResBundle` instance in each file
  of the application that contains translatable strings specifying
  the path where the translated resource files will go or pass a
  `ResBundle` instance around with the string already loaded
    - For react applications, add dependencies on ilib-es6 and
      react-ilib, and then make sure the top level app is wrapped
      in a `LocaleDataProvider` component, so that the `rb` variable
      can be injected to all localizable components
1. Make sure all user-visible strings in the application are wrapped
  in a `ResBundle.getString()` or `ResBundle.getStringJS()` call
    - For react applications, wrap the JSX strings in a
      `Translate` component, and use the `rb.getString()`
      and `rb.getStringJS()` methods for strings in the JS code
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
      a loctool project.json file for the application
1. Run the loctool to extract all of the localizable strings into xliff
  files
1. Send the xliff files out for translation
1. When the translated xliff files are returned, use the loctool again
  to generate the translated resource files in the path that specified
  in the second step above.
1. Add ilib-resource-webpack-loader in the webpack.config.js file with the
  appropriate options (details below)
1. Run webpack as normal, and it should find and include all of the
  translated files that the loctool had just generated into the webpacked
  application

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
batch at the end of the project that of course has to be translated
very quickly to make the project schedule work.

The translated files can be checked in to the source code repo. If
the application includes any file types that are localized by copy,
then it is better to check in the xliff files to the repo and run
the loctool with every build so that translated files are generated
on the fly. The reason for this is that the translated files can
be kept in sync with the source file. Some file
types are localized by extracting strings into a resource file,
which this loader handles, but some other types are localized by creating
translated copies of the source files
with the text in them replaced with translations. For example, HTML files are
localized in this way. If someone changes something non-localizable in
an HTML source file, such as Javascript code or CSS or HTML attributes,
then there will be no new strings to translate and the translated HTML
files can be generated from the source file without any new
translation work. Running the loctool with each build will guarantee
that the non-localizable bits of the translated files will be the
same as the in the source file, even when no strings are changed.
If the application does not have any
file types that are localized by copy and only relies on
resource files, then the translated resource files can be checked in
without this type of problem. File types that are localized by
copy include the following: HTML, XHTML, JSON, or any front-end templates
such as JST, HAML, or JSP. Check with the documentation of the plugin
for the file type to see how it is localized.

# Configuration Details

## webpack.config.js

To add resources to an application, modify the rules section of the
application's webpack.config.js file to add configuration for
the loader:

```
    module: {
        rules: [{
            test: /.js$/,
            use: {
                loader: 'ilib-resource-webpack-loader',
                options: {
                    locales: ["en-US", "de-DE", "ja-JP" ],
                    mode: 'development',
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
  the application needs to support.
- mode - 'development' vs. 'production'
    - in 'development', files are included but not compressed
    - in 'production' mode, files will be compressed/uglified
- resourceDirs - an array of paths where resource files can be found
    - an array allows for including resource files from
      multiple parts of an applications which can be released
      and localized separately
    - paths may be absolute or relative. When relative, they
      should be relative to the root of the applications.

## project.json

In the project.json file, choose the "custom" style of project and
use the loctool plugins that make sense for the application.

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
        "javascript"
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
        "**/*.js"
    ],
    settings: {
        locales: ["es-ES", "de-DE", "fr-FR"],
    }
}
```

In the `plugins` property, list all of the loctool plugins necessary
for all of the file types in the project, and make sure the `resourceDirs`
property contains the output resource directory for each file type. This
should match the `resourceDirs` in the webpack.config.js file.

# Using the Loader and Plugin

Using the loader requires a few things:

- Use npm to install the latest ilib and ilib-resource-webpack-loader locally
- Put ilib-resource-webpack-loader into the webpack.config.js as per above and give them
the appropriate configuration options
- Make sure your code requires or imports ilib classes directly

## Configuration Choices

To configure the loader, you will need to decide upon:

- Which locales does the application need?
- Do you want to assemble the translation data directly into the the app's webpack
bundle, or should they be dynamically lazy-loaded?
- Is the application being built for debug or production mode?

### Which Locales?

The loader is configured by default to support the top 20 locales around the world in
terms of Internet traffic. If no locales are explicitly chosen, the loader will default
to the translations for these top 20 locales. That is a very small subset of all the locales
that could be supported, yet the set of translated strings could still be potentially large.

When the loader is including translations, it first checks if any of the translated
files are available in the resources directory for the given locales. If they are not
there, it does not need to include them, of course.

If the app does not support that many locales, it can have a significantly
smaller footprint by specifying a smaller set of locales in the webpack.config.js.

Locales should be specified using [BCP-47 locale tags](https://en.wikipedia.org/wiki/IETF_language_tag)
(aka. IETF tags). This uses ISO 639 codes for languages, ISO 15924 codes for scripts,
and ISO 3166 codes for regions, separated by dashes. eg. US English is "en-US" and
Chinese for China written with the simplified script is "zh-Hans-CN".

### Assembled or Dynamic Translations?

There are two ways to include the translation data into the webpack configuration for an
application: assembled and dynamic.

1. Assembled. Translations are included into the app's webpack bundle.

   Advantages:

   - everything is loaded and cached at once
   - all translations are available for synchronous use as soon as the browser has
     loaded the js file. No async calls, callbacks, or promises needed!
   - less files to move around and/or to check into your repo

   Disadvantages:

   - that single file can get large if there are a lot of locales (very large!)
   - the app would load all locales at once, even if you only use one locale at a time,
     means extra network bandwidth and load time for data, as well as memory footprint
     that the user isn't actually using

   Assembled translations is a good choice if the app only supports a few locales. The first
   time a user hits the website, they download a larger-than-needed webpack file, but then
   it is cached and everything after that is relatively simple.

2. Dynamic. Webpack has the ability to lazy-load chunks as they are needed. With
   this type of configuration, the code is assembled into a single file, but the translation
   data goes into separate webpack chunks, and webpack lazy-loads those chunks on the
   fly as they are needed.

   Advantages:

   - file size and therefore initial page load time are minimized
   - the application code can be cached in the browser and doesn't change often even if
     the translations do
   - the translation files can be cached separately, allowing the app to add new locales
     to the web site later without affecting any existing cache for the translations in
     other locales

   Disadvantages:

   - the number of locale translation files can get unwieldy if the app has a lot of locales
   - since webpack loads the bundles asynchronously, the app must use the ilib ResBundle class
     asynchronously with callbacks or promises. Alternately, the app must pre-initialize the
     translation data asynchronously and then wait for the load to finish in a jquery
     "document ready" kind of callback function before using the translations synchronously
     after that.

   Using dynamic data is a good choice if you have a lot of locales or have a lot of strings
   to translate.














### Compressed or Not?

You can compress/uglify ilib code normally using the regular uglify webpack plugin.
Note that the ilib code in npm is already uglified, so you will get that by default
(except for the webpack glue/wrapper code of course.)

If you would like to create a version of ilib that is NOT compressed, you're going
to have to do a little more work. See the sections below for details on how to
do that.

## What Do the Loader and Plugin Do?

Webpack loaders in general are used to read and modify javascript files as they are
added to the webpack bundle. The ilib-webpack-loader does this as well. Specifically,
it is doing three things:

* Read javascript files as they are added to the bundle to find any references to
iLib classes. These iLib classes are added to the list of classes that will
to be assembled into the webpack build later and for which locale data is needed.
In this way, you only get the data for those iLib classes that are actually used.
* If the javascript file being added to the bundle happens to be one of the iLib
classes, search the file looking for special comments that document exactly which
types of locale data that class needs. Each iLib class is self-documenting. The
list of locale data types is saved for later as well.
* Also, if the file being added is an iLib class, other special comments
are replaced with bits of code depending on your configuration. This helps the
code operate properly in all configurations.

Webpack plugins in general are used to run custom code at various points in the
webpack bundling process. The ilib-webpack-plugin hooks into the process at the
point after all of the javascript files
are already added to the webpack bundle. At that point, it knows the final set
of iLib classes that are in use in that bundle, and the list of locale data
types that those classes need. It can combine that info with the configured
set of locales to write out only the locale data that is actually needed by your
project. The data for each locale is either assembled in to the single webpack
bundle for "assembled" mode, or it is written out into separate files that become
their own webpack bundles for "dynamicdata" mode. This minimizes the size of
the locale data in both modes, and also allows the data to be loaded dynamically
in "dynamicdata" mode.

## Using the Loader and Plugin in Your Own Webpack Config

If you would like to use the loader and plugin in your own webpack configuration
to create a minimal ilib, here are the steps:

1. Install ilib, the loader, and the plugin from npm:

  ```
  npm install --save-dev ilib ilib-webpack-loader ilib-webpack-plugin
  ```

1. Examine the section above on configuration choices and then choose your locales
and data loading style

1. In your webpack configuration, update the rules section like this:

   ```javascript
        // this goes outside the module.export
        let options = {
            locales: ["en-US", "de-DE", "fr-FR", "it-IT", "ja-JP", "ko-KR", "zh-Hans-CN"],
            assembly: "dynamicdata",
            compilation: 'compiled',
            size: 'custom',
            tempDir: 'assets'
        };

        ...

        // inside the config:
        module: {
            rules: [{
                test: /\.(js|html)$/, // Run this loader on all .js and .html files, even non-ilib ones
                use: {
                    loader: "ilib-webpack-loader",
                    options: options
                }
            }]
        },
   ```

   and the plugins section like this:

   ```javascript
        plugins: [
            new IlibWebpackPlugin(options)
        ],
   ```

   The locales options to the loader and plugin are self-explanatory. Make sure to pass the same set of
   locales to both by defining the options object earlier in the file. If you pass different options to
   the loader and plugin, you will get inconsistent results!

   The assembly option can be one of "assembled" or "dynamicdata".

   The compilation option can be one of "compiled" or "uncompiled".

   The size option should always be set to "custom" when you are creating your own version of iLib. If
   you choose any of the pre-assembled sizes, you will get a fixed set of iLib classes, which is probably
   not what you want.

   The tempDir option is a directory where the ilib loader and plugin can write out the extracted/filtered
   locale data files so that they can be included into the webpack bundle. Make sure this directory is
   outside of your bundle output path.

1. Put ilib into its own vendor bundle:

   ```javascript
   module.exports = [{
       // your regular app configuration here
   }, {
       // ilib bundle entry point here
       entry: "ilib/lib/ilib.js",
       output: {
           filename: 'ilib.js',
           chunkFilename: 'ilib.[name].js',  // to name the locale bundles
           path: outputPath,                 // choose an appropriate output dir
           publicPath: urlPath,              // add the corresponding URL
           library: 'ilib',
           libraryTarget: 'umd'
       },
       module: {
           rules: [{
               test: /\.(js|html)$/,        // Run this loader on all .js or .html files
               use: {
                   loader: "ilib-webpack-loader",
                   options: options
               }
           }]
           ...
       },
       plugins: [
            new IlibWebpackPlugin(options)
       ],
   }];
   ```

   If you want, you can change the name of the ilib bundle file by changing the output.filename property,
   or the name of the locale bundle files by changing the `output.chunkFileName`
   property. For example, you might like to include the hash into the file name for cache busting
   purposes. See the [webpack documentation](https://webpack.js.org/configuration/output/#output-chunkfilename)
   on the file name template rules.

   The path property should point to the directory where you want the output files to go, and the publicPath
   property should be the sub-URL under your web server where the ilib files will live. Webpack
   uses the URL to load the locale bundle files dynamically via XHR. For example, if your js files are located
   at https://www.mycompany.com/project/assets/javascript/*.js, then you would set the publicPath to
   "/project/assets/javascript". Webpack automatically loads files from the same server, so you don't need to
   specify the server part of the URL, nor the "https://" part.

1. Include a special ilib file at the start of your app that is used to load all of the iLib classes and
locale data:

   ```javascript
   // this code installs the components that know how to load the locale data:
   const ilibdata = require("ilib/lib/ilib-getdata.js");
   ```

1. Make your code require or import ilib classes directly:

   ```javascript
   const DateFmt = require("ilib/lib/DateFmt");
   ```

   or, under ES6, use import from the ilib-es6 wrappers project instead:

   ```javascript
   import DateFmt from "ilib-es6/lib/DateFmt";
   ```

# How it Works

The loader operates by examining all js (or html) files that are in your webpack configuration
as webpack processes them. For regular javascript files, it searches them looking for
references to any iLib classes. If the javascript file being loaded happens to be an iLib
class, the loader will search it looking for a special comment that documents exactly which
types of locale data this class needs. Additionally, in dynamicdata mode, the loader will
generate a set of empty locale data files that get added to the bundle.

At the end of the loading, the plugin runs. In assembled mode, it will make sure that
all of the required locale data files for the data types and locales are added to the
bundle. In dynamic data mode, it will fill in actual contents into the locale data
files that the loader previously created and make sure webpack generates bundles for
each of those files.

Step-by-step:

1. Run webpack in your app as normal.
1. Webpack processes each of your js files looking for calls to ilib.
1. The ilib-webpack-loader also processes each of your js files looking for special
   comments that indicate that they use certain types of locale data. For example,
   the DateFmt class has this comment in it:

   ```javascript
   // !data dateformats
   ```

   This indicates that it uses the various `dateformats.json` files in the ilib locale
   directory. (Look in ...node_modules/ilib/js/data/locale/* if you're curious.) You can
   use these comments in your own code if you need to load in extra non-locale data files
   such as character sets, character mapping files, or time zones that are not
   automatically included by the loader.
1. When the all of the loaders are finished running, the ilib-webpack-plugin will emit
   the locale data files to disk. In the example above, if the locales are set to "en-US" and
   "fr-FR", the information from the dateformat.json files from the previous point
   would go into:

   ```
   ilib/js/locale/dateformats.json -> root.js
   ilib/js/locale/en/dateformats.json -> en.js
   ilib/js/locale/en-US/dateformats.json -> en-US.js
   ilib/js/locale/und/US/dateformats.json -> und-US.js
   ilib/js/locale/fr/dateformats.json -> fr.js
   ilib/js/locale/fr-FR/dateformats.json -> fr-FR.js
   ilib/js/locale/und/FR/dateformats.json -> und-FR.js
   ```

   In assembled mode, the loader adds require() calls for these 7 new files so that
   they are included directly into the ilib bundle. Alternately, in dynamicdata mode,
   it would add calls to System.import() for each of them which causes
   webpack to issue each file as its own separate bundle that can be loaded
   dynamically.
1. Webpack will emit a number of files in the output directory:

   ```
   my-app.js    - your own bundle
   ilib.js      - the ilib code bundle, which you can put in a script tag in your html
   ilib.root.js - the 7 locale data bundles (in dynamicdata mode only), which all go onto your web server as well
   ilib.en.js
   ilib.en-US.js
   ilib.und-US.js
   ilib.fr.js
   ilib.fr-FR.js
   ilib.und-FR.js
   ```

Why so Many Locale Data Files?
-----

You may be wondering why there are so many locale data files/bundles emitted in the
example above when the configuration only requested 2 locales. The loader could have
just emitted two bundles `ilib.en-US.js` and `ilib.fr-FR.js`, right?

The answer is footprint. By splitting the files, each piece of locale data is included
only once. For example, the root.js contains a lot of non-locale data that does not need to be
replicated in each of those two files. For example, the Unicode character type properties that the
CType functions use are the same for all locales. Each Unicode character is unambiguous
and does not depend on which locale you are using. A digit is always a digit. Why have two
copies of that info on your web server? If the user's browser loads `ilib.root.js` once,
it can cache it and not load it again, no matter the locale. This gets even more important
when you have more than 2 locales at once.

Similarly, if your configuration specifies multiple English locales (maybe your app
supports all of these: en-US, en-CA, en-GB, and en-AU), then
the common English data does not need to be replicated in each of those files. The
`ilib.en.js` bundle will contain the shared settings that are common to many varieties
of English, and the file `ilib.en-US.js` only contains those locale data and settings
that truly specific to English as spoken in the US. The en-US file is much smaller than
the en file.

Creating an Uncompressed Version of iLib
---

By default all of the ilib code published to npm is uglified already. That means that
any ilib classes you include will appear in your bundles as uglified. If you are trying
to do debugging, maybe you want to use the uncompressed version of ilib instead?

In order to make an uncompressed version of ilib, follow these steps:

1. Clone the ilib repo from [github](https://github.com/iLib-js/iLib).

1. Install java 1.8 and ant to build it. Yes, we will be moving
   to grunt soon to build the js parts. The ilib repo also includes some
   java code for Android, so we have to keep Java and ant for now.

1. cd to the "js" directory and enter "ant". Allow it to build some stuff.

Now you can point your webpack configuration to this freshly built ilib, which
contains the uncompressed code and locale data files. Here are the changes.

```javascript
    // this goes outside the module.exports
    let options = {
        locales: ["en-US", "de-DE", "fr-FR", "it-IT", "ja-JP", "ko-KR", "zh-Hans-CN"],
        assembly: "dynamicdata",
        compilation: "uncompiled",           // <- These are the new parts!
        ilibRoot: "full/path/to/your/ilib/clone", // <- These are the new parts!
        size: 'custom',
        tempDir: 'assets'
    };

    module.exports = [{
        // your regular app configuration here
    }, {
       // ilib bundle entry point here
       entry: "full/path/to/your/ilib/clone/js/lib/ilib.js",
       output: {
           filename: 'ilib.js',
           chunkFilename: 'ilib.[name].js',  // to name the locale bundles
           path: outputPath,                 // choose an appropriate output dir
           publicPath: "/" + urlPath,        // add the corresponding URL
           library: 'ilib',
           libraryTarget: 'umd'
       },
       module: {
           rules: [{
               test: /\.(js|html)$/,        // Run this loader on all .js or .html files
               use: {
                   loader: "ilib-webpack-loader",
                   options: options
               }
           }]
           ...
       },
       plugins: [
            new IlibWebpackPlugin(options)
       ],
   }];
```

Note that the "entry" property has changed from the examples above, and there
is a new value for the "compilation" option passed to the loader. Also, a new
parameter "ilibRoot" points to the root of the iLib clone.


# What if my Website Project is not Currently Using Webpack?

You can still use webpacked ilib! If you have javascript in js and html
files, but you currently don't use webpack for your own project, you have
have two choices:

1. Use a standard build of ilib from the [ilib releases page on github](https://github.com/iLib-js/iLib/releases).

1. Build your own customized version of ilib

Using Standard Builds
-----

You can use a pre-built version of ilib based on releases published on
[the ilib project's releases page on github](https://github.com/iLib-js/iLib/releases).

Look inside ilib-&lt;version>.tgz or ilib-&lt;version>.zip for the standard
builds.

Releases of ilib come with three
pre-built sizes: core, standard, and full. The core size includes a minimal
set of classes that pretty much only allows you to do simple things like
translating text. The standard size has all the basics such as date formatting
and number formatting, as well as text translation and a few other classes.
The full size has every class that ilib contains.

Releases also now come with the fully assembled and dynamicdata versions of
each size for web sites or node. The locale data that comes with each is for the
top 20 locales on the Internet by volume of traffic.

For fully dynamic code and locale data loading for use with nodejs or rhino/nashorn,
you can install the latest ilib from npm.

Using a standard release of ilib is convenient, but it may not contain the locale
data you need and/or the classes you need, or it may be too large with too many
locales. If that's the case for your project, you can build a custom version of
ilib that contains only the code and data you actually need and use.

Creating a Custom Version of iLib
----

If you do not use webpack in your own project, but you would still like to create a custom
version of ilib that includes only the code and data that your app needs, you can do
that! Here is an example of how:

First, let's assume you have a web app which supports English for the US, and French for
France. Also, we assume that you have installed ilib, webpack, ilib-webpack-loader, and
ilib-webpack-plugin via npm.

1. A sister module `ilib-scanner` contains a node-based tool that can scan your code looking
   for references to ilib classes. If your node_modules/.bin
   directory is in your path, you can execute this tool directly on the command-line. This tool
   will generate both an ilib metafile that will include only the classes you need,
   and a webpack.config.js file that configures webpack to create that customized ilib.js file.

1. Change directory to the root of your web app, and run `ilib-scanner` with the following options:

   ```
   ilib-scanner --assembly=assembled --locales=en-US,fr-FR --compilation=compiled ilib-include.js
   ```

   The "assembly" parameter can have the value of either "assembled" and "dynamicdata". Default is
   "assembled".

   The value of the locales parameter is a comma-separated list of locales that your app needs
   to support. In our example, this is en-US for English/US and fr-FR for French/France.

   The "compilation" parameter is one of "compiled" or "uncompiled".

   You must give the path to the metafile file you would like to generate. In this
   example, that is "ilib-include.js". The scanner will fill this file with explicit "require"
   calls for any ilib class your code uses.

   Optionally you can follow the metafile name with a list of the paths to
   you would like to scan.  Without those
   explicit paths, the default is to recursively scan the current directory looking for js
   and html files.

   When the tool is done, the new files are generated in the same path that you gave to
   the metafile. So for example, if you gave the metafile path output/js/ilib-include.js, then
   the output files will be output/js/ilib-include.js and output/js/webpack.config.js.

1. Examine the webpack.config.js file to make sure the settings are appropriate. You can do things
   like change the name of the ilib output file (`output.filename` property) if desired. It should
   be set up to generate a file called ilib.js properly already, so you don't have to modify
   anything.

   If you have requested a dynamicdata build, you must make sure the `output.publicPath`
   property is set to the directory part of the URL where webpack can load the locale data
   files. For example, if you put ilib and the locale data files underneath
   "http://www.mycompany.com/scripts/js/ilib.js", then set the publicPath property to "/scripts/js/".
   Webpack uses XHR requests to the server where it loaded ilib.js from in order to load the
   corresponding locale data files under the path given in the publicPath directory.

1. Run "webpack" in the dir with the new webpack.config.js in it. It will churn for a while and
   then spit out files in the path
   named in the webpack.config.js. By default, the file name is "ilib.js".

1. Update your html files to include the new custom build of ilib with a standard script tag:

    ```html
    <script src="/path/to/ilib.js"></script>
    <script>
       // All of the classes have been copied to the global scope here, so
       // you can just start using them:
       new DateFmt({
           locale: "fr-FR",
           sync: false,
           onLoad: function(df) {
               alert("Aujourd'hui, c'est " + df.format(new Date()));
           }
       });
    </script>
    ```

Et voila. You are done.

Note that ilib automatically copies its public classes up to the global scope,
so you can just use them normally, not as a property of the "ilib" namespace.
If you used ilib 12.0 or earlier, this is the same as how it worked before, so
if you are upgrading to 13.0 or higher, you will probably
not need to change your code. If you don't want to pollute your global scope,
you can use all of the classes via the ilib namespace. Just remove the
require call for "ilib-unpack.js" in the generated metafile and rerun webpack.

Now upload the ilib.js (and for dynamicdata mode, all of the locale data
files as well) to your web server or check it in to your
repo so that it all gets published with the next push. We also recommend that
you check these files in to your source code control system.

# Examples

## Simple Example

All of the code from the snippets above:

webpack.config.js:

```javascript
var path = require("path");

var IlibWebpackPlugin = require("ilib-webpack-plugin");

var options = {
    // edit these for the list of locales you need
    locales: ["en-US", "fr-FR", "de-DE", "ko-KR"],
    assembly: "dynamicdata",
    compilation: "uncompiled",
    tempDir: 'assets'
};

module.exports = {
    // ilib bundle entry point here
    entry: path.resolve("./ilib-metafile.js"),
    output: {
        filename: 'ilib-custom.js',       // you can change this if you want
        chunkFilename: 'ilib.[name].js',  // to name the locale bundles
        path: path.resolve("./output"),   // choose an appropriate output dir
        publicPath: "output/",          // choose the URL where ilib will go
        library: 'ilib',
        libraryTarget: 'umd'
    },
    module: {
        rules: [{
            test: /\.(js|html)$/,        // Run this loader on all .js files
            use: {
                loader: "../ilib-webpack-loader.js",
                options: options
            }
        }]
    },
    plugins: [
        new IlibWebpackPlugin(options)
    ]
};
```

ilib-metafile.js:

```html
var ilib = require("ilib/lib/ilib.js");

// assign each class to a subproperty of "ilib"
ilib.Locale = require("ilib/lib/Locale.js");
ilib.DateFmt = require("ilib/lib/DateFmt.js");
ilib.NumFmt = require("ilib/lib/NumFmt.js");

// This unpacks the above classes to the global scope
require("ilib/lib/ilib-unpack.js");

// Must be at the end of meta file to generate the locale data files
require("ilib/lib/ilib-getdata.js");

module.exports = ilib;
```

index.html:

```javascript
<html>
<head>
<meta charset="UTF-8">
<script src="output/ilib-custom.js"></script>
<script>
    // all of the classes have been copied to the global scope here, so
    // you can just start using them:
    new DateFmt({
        locale: "fr-FR",
        sync: false,
        onLoad: function(df) {
            alert("Aujourd'hui, c'est " + df.format(new Date()));
        }
    });
</script>
</head>
<body>
Test page
</body>
</html>
```

The above example code is also located in examples subdirectory of the ilib-webpack-loader
clone so you can try it
for yourself. Just change dir into `examples` and run "webpack" with no arguments.

The example above is written with an asynchronous
call to the DateFmt constructor, so you can try changing the `assembly` property in the
webpack.config.js to `dynamicdata`, run webpack again, reload the html, and it should
still work properly. You will see on the console that the packages for French have been
loaded dynamically and that the date appears with a French format (dd/MM/yyyy) in the
alert dialog.

## Example of a Customized Build

A working example of a customized version of ilib for a site that does not currently
use webpack can be found in the ilib demo app. This is included
in the ilib sources under the docs/demo directory. See
[the ilib demo app on github](https://github.com/iLib-js/iLib/tree/development/docs/demo)
for details. You can try it out for yourself if you git clone the ilib project,
change directory to ilib/docs/demo and then use the instructions above to create
a customized version of ilib for [projects that are not currently using
webpack](#what-if-my-website-project-is-not-currently-using-webpack).



                                                 Fin.
