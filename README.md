# buseref [![Build Status](https://travis-ci.org/binarygeotech/buseref.svg?branch=master)](https://travis-ci.org/binarygeotech/buseref) [![Coverage Status](https://coveralls.io/repos/github/binarygeotech/buseref/badge.svg?branch=master)](https://coveralls.io/github/binarygeotech/buseref?branch=master)

[![NPM](https://nodei.co/npm/buseref.png?downloads=true)](https://nodei.co/npm/buseref/)

> Parse build blocks in HTML files to replace references

Forked from the Jonathan Kemp [useref](https://github.com/jonkemp/useref/) with more options.

## Installation

```
npm install buseref
```

## Usage

```js
var useref = require('buseref');
var result = useref(inputHtml);
// result = [ replacedHtml, { type: { path: { 'assets': [ replacedFiles] }}} ]
```
or

```js
var useref = require('buseref');
var result = useref.file(filename);
// result = [ replacedHtml, { type: { path: { 'assets': [ replacedFiles] }}} ]
```

Blocks are expressed as:

```html
<!-- build:<type>(alternate search path) <path> <parameters> -->
... HTML Markup, list of script / link tags.
<!-- endbuild -->
```

- **type**: either `js`, `css` or `remove`
- **alternate search path**: (optional) By default the input files are relative to the treated file. Alternate search path allows one to change that
- **path**: the file path of the optimized file, the target output
- **parameters**: extra parameters that should be added to the tag

An example of this in completed form can be seen below:

```html
<html>
<head>
  <!-- build:css css/combined.css -->
  <link href="css/one.css" rel="stylesheet">
  <link href="css/two.css" rel="stylesheet">
  <!-- endbuild -->
</head>
<body>
  <!-- build:js scripts/combined.js -->
  <script type="text/javascript" src="scripts/one.js"></script>
  <script type="text/javascript" src="scripts/two.js"></script>
  <!-- endbuild -->

  <!-- build:js scripts/async.js async data-foo="bar" -->
  <script type="text/javascript" src="scripts/three.js"></script>
  <script type="text/javascript" src="scripts/four.js"></script>
  <!-- endbuild -->
</body>
</html>
```

The module would be used with the above sample HTML as follows:

```js
var result = useref(sampleHtml);

// [
//   resultHtml,
//   {
//     css: {
//       'css/combined.css': {
//         'assets': [ 'css/one.css', 'css/two.css' ]
//       }
//     },
//     js: {
//       'scripts/combined.js': {
//         'assets': [ 'scripts/one.js', 'scripts/two.js' ]
//       },
//       'scripts/async.js': {
//          'assets': [ 'scripts/three.js', 'scripts/four.js' ]
//        }
//     }
//   }
// ]
```


The resulting HTML would be:

```html
<html>
<head>
  <link rel="stylesheet" href="css/combined.css"/>
</head>
<body>
  <script src="scripts/combined.js"></script>
  <script src="scripts/async.js" async data-foo="bar" ></script>
</body>
</html>
```

## IE Conditional Comments

Internet Explorer Conditional Comments are preserved. The code below:

```html
<!-- build:js scripts/combined.js   -->
<!--[if lt IE 9]>
<script type="text/javascript" src="scripts/this.js"></script>
<script type="text/javascript" src="scripts/that.js"></script>
<![endif]-->
<!-- endbuild -->
```

Results in:

```html
<!--[if lt IE 9]>
<script src="scripts/combined.js"></script>
<![endif]-->
```

### Custom blocks

Sometimes you need a bit more. If you would like to do custom processing, this is possible with a custom block, as demonstrated below.

```html
<!-- build:import components -->
<link rel="import" href="/bower_components/some/path"></link>
<!-- endbuild -->
```

With

```js
var useref = require('buseref');
var result = useref(inputHtml, {
  // each property corresponds to any blocks with the same name, e.g. "build:import"
  import: function (content, target, options, alternateSearchPath) {
    // do something with `content` and return the desired HTML to replace the block content
    return content.replace('bower_components', target);
  }
});
```

Becomes

```html
<link rel="import" href="/components/some/path"></link>
```

The handler function gets the following arguments:

- *content* (String): The content of the custom use block
- *target* (String): The "path" value of the use block definition
- *options* (String): The extra attributes from the use block definition, the developer can parse as JSON or do whatever they want with it
- *alternateSearchPath* (String): The alternate search path that can be used to maintain a coherent interface with standard handlers

Include a handler for each custom block type.

### Symfony Twig and Laravel 5 Blade assets

Works with the [symfony2 assetic](http://symfony.com/doc/current/cookbook/assetic/asset_management.html) and [laravel asset](https://laravel.com/docs/5.1/helpers#method-asset) and [elixir](https://laravel.com/docs/5.2/elixir#versioning-and-cache-busting) links in twig or blade or html or php.

```html
<!-- build:js scripts/combined.js -->
<script src="{{ asset('symfony/js/script.js') }}"></script>
<script src="{{ elixir('laravel/js/script.js') }}"></script>
<!-- endbuild -->
```

### Options

#### options.noconcat

Type: `Boolean`  
Default: `false`  

Strips out build comments but leaves the rest of the block intact without replacing any tags.

```html
<!-- build:js scripts/combined.js   -->
<script type="text/javascript" src="scripts/this.js"></script>
<script type="text/javascript" src="scripts/that.js"></script>
<!-- endbuild -->
```

Results in:

```html
<script type="text/javascript" src="scripts/this.js"></script>
<script type="text/javascript" src="scripts/that.js"></script>
```
#### options.render

Type: `Boolean`  
Default: `true`  

If value is ```false```, it strips out build comments and replacing the block without rendering any ```link``` or ```script``` tags.
Only useful with ```css``` and ```js``` blocks.

```html
<!-- build:js scripts/combined.js   -->
<script type="text/javascript" src="scripts/this.js"></script>
<script type="text/javascript" src="scripts/that.js"></script>
<!-- endbuild -->
```

Results in:

```html
scripts/combined.js
```
#### options.callback

Type: `Function`  
Default: `null`  

If you would like to do handle the processing of all blocks, this is possible with the callback, as demonstrated below.

```html
<!-- build:css combined.css -->
<link href="css/one.css" rel="stylesheet">
<link href="css/two.css" rel="stylesheet">
<!-- endbuild -->
```

With

```css
var useref = require('buseref');
var result = useref(inputHtml, {
  callback: function (content, type, target, attributes) {
    // do something with `content` and return the desired HTML to replace the block content
    // types: css, js or remove
    
    var ref = '';
    
    if (type === 'css') {
      // css link element regular expression
      // TODO: Determine if 'href' attribute is present.
      var regcss = /<?link.*?(?:>|\))/gmi;

      // Check to see if there are any css references at all.
      if (content.search(regcss) !== -1) {
        if (attributes) {
          ref = '<link rel="stylesheet" href="' + target + '" ' + attributes + '>';
        } else {
          ref = '<link rel="stylesheet" href="' + target + '">';
        }
      }
    }
    
    return ref;
  }
});
```

Becomes

```html
<link rel="stylesheet" href="combined.css">
```

The handler function gets the following arguments:

- *content* (String): The content of the custom use block
- *type* (String): The block type; css, js or remove
- *target* (String): The "path" value of the use block definition
- *attibutes* (String): The extra attributes from the use block definition, the developer can parse as JSON or do whatever they want with it

## Contributing

See the [CONTRIBUTING Guidelines](https://github.com/binarygeotech/buseref/blob/master/CONTRIBUTING.md)

## License

MIT © [Okojie Davis GEORGE](http://okojiedgeorge.com)
