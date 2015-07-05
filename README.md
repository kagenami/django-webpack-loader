# django-webpack-loader

[![Join the chat at https://gitter.im/owais/django-webpack-loader](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/owais/django-webpack-loader?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://travis-ci.org/owais/django-webpack-loader.svg?branch=master)](https://travis-ci.org/owais/django-webpack-loader)
[![codecov.io](http://codecov.io/github/owais/django-webpack-loader/coverage.svg?branch=master)](http://codecov.io/github/owais/django-webpack-loader?branch=master)

<br>
Read http://owaislone.org/blog/webpack-plus-reactjs-and-django/ for a detailed step by step guide on setting up webpack with django using this library.

<br>

Use webpack to generate your static bundles without django's staticfiles or opaque wrappers.


Django webpack loader consumes the output generated by [webpack-bundle-tracker](https://github.com/owais/webpack-bundle-tracker) and lets you use the generated bundles in django.


<br>
## Compatibility

Test cases cover Django>=1.6 on Python 2.7 and Python>=3.3. 100% code coverage is the target so we can be sure everything works anytime. It should probably work on older version of django as well but the pacakge does not ship any test cases for them.


## Install

```bash
npm install --save-dev webpack-bundle-tracker

pip install django-webpack-loader
```

<br>
## Configuration

<br>
### Assumptions

Assuming `BASE_DIR` in settings refers to the root of your django app.
```python
import sys
import os

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
```

<br>
Assuming `assets/` is in `settings.STATICFILES_DIRS` like

```python
STATICFILES_DIRS = (
    os.path.join(BASE_DIR, 'assets'),
)
```

<br>
Assuming your webpack config lives at `./webpack.config.js` and looks like this
```javascript
module.exports = {
  context: __dirname,
  entry: {
    app: './assets/js/index'
    output: {
      path: require('path').resolve('./assets/bundles/'),
      filename: '[name]-[hash].js',
  },

  plugins: [
    new BundleTracker({filename: './webpack-stats.json'})
  ]
}

```


<br>
### Default Configuration
```python
WEBPACK_LOADER = {
    'BUNDLE_DIR_NAME': 'webpack_bundles/', # must end with slash
    'STATS_FILE': 'webpack-stats.json',
    'POLL_DELAY': 0.2,
    'IGNORE': ['.+\.hot-update.js', '.+\.map']
}
```

<br>

#### BUNDLE_DIR_NAME
```python
WEBPACK_LOADER = {
	'BUNDLE_DIR_NAME': 'bundles/' # end with slash
}
```

`BUNDLE_DIR_NAME` refers to the dir in which webpack outputs the bundles. It should not be the full path. If `./assets` is one of you static dirs and webpack generates the bundles in `./assets/output/bundles/`, then `BUNDLE_DIR_NAME` should be `output/bundles/`.

If the bundle generates a file called `main-cf4b5fab6e00a404e0c7.js` and your STATIC_URL is `/static/`, then the `<script>` tag will look like this

```html
<script type="text/javascript" src="/static/output/bundles/main-cf4b5fab6e00a404e0c7.js"/>
```

<br>

#### STATS_FILE
```python
WEBPACK_LOADER = {
	'STATS_FILE': os.path.join(BASE_DIR, 'webpack-stats.json')
}
```

`STATS_FILE` is the filesystem path to the file generated by `webpack-bundle-tracker` plugin. If you initialize `webpack-bundle-tracker` plugin like this

```javascript
new BundleTracker({filename: './webpack-stats.json'})
```

and your webpack config is located at `/home/src/webpack.config.js`, then the value of `STATS_FILE` should be `/home/src/webpack-stats.json`

<br>

#### IGNORE
`IGNORE` is a list of regular expressions. If a file generated by webpack matches one of the expressions, the file will not be included in the template.

<br>

#### POLL_INTERVAL

`POLL_INTERVAL` is the number of seconds webpack_loader should wait between polling the stats file. The stats file is polled every 200 miliseconds by default and any requests to are blocked while webpack compiles the bundles. You can reduce this if your bundles take shorter to compile.

**NOTE:** Stats file is not polled when in production (DEBUG=False).

<br>


## Usage
<br>

#### Manually run webpack to build assets.

One of the core principles of django-webpack-loader is to not manage webpack itself in order to give you the flexibility to run webpack the way you want. If you are new to webpack, check one of the [examples](https://github.com/owais/django-webpack-loader/tree/master/examples), read [my detailed blog post](http://owaislone.org/blog/webpack-plus-reactjs-and-django/) or check [webpack docs](http://webpack.github.io/).

#### Settings

Add `webpack_loader` to `INSTALLED_APPS`

```
INSTALLED_APPS = (
    ...
    'webpack_loader',
)
```

#### Templates

```HTML+Django
{% load render_bundle from webpack_loader %}

{% render_bundle 'main' %}
```

`render_bundle` will render the proper `<script>` and `<link>` tags needed in your template.

`render_bundle` also takes a second argument which can be a file extension to match. This is useful when you want to render different types for files in separately. For example, to render CSS in head and JS at bottom we can do something like this,

```HTML+Django
{% load render_bundle from webpack_loader %}

<html>
  <head>
    {% render_bundle 'main' 'css' %}
  </head>
  <body>
    ....
    {% render_bundle 'main' 'js' %}
  </body>
</head>
```


<br>

## How to use in Production

**It is up to you**. There are a few ways to handle this. I like to have slightly separate configs for production and local. I tell git to ignore my local stats + bundle file but track the ones for production. Before pushing out newer version to production, I generate a new bundle using production config and commit the new stats file and bundle. I store the stats file and bundles in a directory that is added to the `STATICFILES_DIR`. This gives me integration with collectstatic for free. The generated bundles are automatically collected to the target directory and synched to S3.


`./webpack_production.config.js`
```javascript
config = require('./webpack.config.js');

config.output.path = require('path').resolve('./assets/dist');

config.plugins = [
    new BundleTracker({filename: './webpack-stats-prod.json'})
]

// override any other settings here like using Uglify or other things that make sense for production environments.

module.exports = config;
```

`settings.py`
```python
if not DEBUG:
    WEBPACK_LOADER.update({
        'BUNDLE_DIR_NAME': 'dist/',
        'STATS_FILE': os.path.join(BASE_DIR, 'webpack-stats-prod.json'
    })
```

<br><br>


You can also simply generate the bundles on the server before running collectstatic if that works for you.

## Extra

### Jinja2 Configuration

If you need to output your assets in a jinja template we provide a Jinja2 extension that's compatible with the [Django Jinja](https://github.com/niwinz/django-jinja) module and Django 1.8.

To install the extension add it to the django_jinja `TEMPLATES` configuration in the `["OPTIONS"]["extension"]` list.

```python
TEMPLATES = [
    {
        "BACKEND": "django_jinja.backend.Jinja2",
        "OPTIONS": {
            "extensions": [
                "django_jinja.builtins.extensions.DjangoFiltersExtension",
                "webpack_loader.contrib.jinja2ext.WebpackExtension",
            ],
        }
    }
]
```

Then in your base jinja template:

```HTML+Jinja2
{{ render_bundle('main') }}
```

--------------------
<br>


Enjoy your webpack with django :)
