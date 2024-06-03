# Django Vite

[![PyPI version](https://badge.fury.io/py/django-vite.svg)](https://badge.fury.io/py/django-vite)

Integration of [ViteJS](https://vitejs.dev/) in a Django project.

- [Installation](#installation)
  - [Django](#django)
  - [ViteJS](#vitejs)
  - [Assets](#assets)
- [Usage](#usage)
  - [Configuration](#configuration)
  - [Dev Mode](#dev-mode)
  - [Template tags](#template-tags)
  - [Custom attributes](#custom-attributes)
  - [Jinja2 template backend](#jinja2-template-backend)
- [Vite Legacy Plugin](#vite-legacy-plugin)
- [Multi-app configuration](#multi-app-configuration)
- [Configuration Variables](#configuration-variables)
  - [dev\_mode](#dev_mode)
  - [dev\_server\_protocol](#dev_server_protocol)
  - [dev\_server\_host](#dev_server_host)
  - [dev\_server\_port](#dev_server_port)
  - [static\_url\_prefix](#static_url_prefix)
  - [manifest\_path](#manifest_path)
  - [legacy\_polyfills\_motif](#legacy_polyfills_motif)
  - [ws\_client\_url](#ws_client_url)
  - [react\_refresh\_url](#react_refresh_url)
- [Notes](#notes)
  - [Whitenoise](#whitenoise)
- [Examples](#examples)
- [Thanks](#thanks)


## Installation

### Django

```
pip install django-vite
```

Add `django_vite` to your `INSTALLED_APPS` in your `settings.py`
(before your apps that are using it).

```python
INSTALLED_APPS = [
    ...
    'django_vite',
    ...
]
```

### ViteJS

Follow instructions on [https://vitejs.dev/guide/](https://vitejs.dev/guide/).
And mostly the SSR part.

Then in your ViteJS config file :

- Set the `base` options the same as your `STATIC_URL` Django setting.
- Set the `build.outDir` path to where you want the assets to compiled.
- Set the `build.manifest` options to `manifest.json`.
- As you are in SSR and not in SPA, you don't have an `index.html` that
  ViteJS can use to determine which files to compile. You need to tell it
  directly in `build.rollupOptions.input`.

```javascript
export default defineConfig({
  ...
  base: "/static/",
  build: {
    ...
    manifest: "manifest.json",
    outDir: resolve("./assets"),
    rollupOptions: {
      input: {
        <unique key>: '<path to your asset>'
      }
    }
  }
})
```

### Assets

As recommended on Vite's [backend integration guide](https://vitejs.dev/guide/backend-integration.html), your assets should include the modulepreload polyfill.

```javascript
// Add this at the beginning of your app entry.
import 'vite/modulepreload-polyfill';
```

## Usage

### Configuration

Define a default `DJANGO_VITE` configuration in your `settings.py`.

```python
DJANGO_VITE = {
  "default": {
    "dev_mode": True
  }
}
```

Or if you prefer to use the legacy module-level settings, you can use:

```python
DJANGO_VITE_DEV_MODE = True
```

Be sure that the `build.outDir` from `vite.config.js` is included in `STATICFILES_DIRS`.

```python
STATICFILES_DIRS = [
  BASE_DIR / "assets"
]
```

### Dev Mode

The `dev_mode`/`DJANGO_VITE_DEV_MODE` boolean defines if you want to include assets in development mode or production mode.
- In development mode, assets are included as modules using the ViteJS
  webserver. This will enable HMR for your assets.
- In production mode, assets are included as standard assets
  (no ViteJS webserver and HMR) like default Django static files.
  This means that your assets must be compiled with ViteJS before.
- This setting may be set as the same value as your `DEBUG` setting in
  Django. But you can do what is good for your needs.

### Template tags

Include this in your base HTML template file.

```
{% load django_vite %}
```

Then in your `<head>` element add this :

```
{% vite_hmr_client %}
```

- This will add a `<script>` tag to include the ViteJS HMR client.
- This tag will include this script only if `DJANGO_VITE_DEV_MODE` is true,
  otherwise this will do nothing.

Then add this tag (in your `<head>` element too) to load your scripts :

```
{% vite_asset '<path to your asset>' %}
```

This will add a `<script>` tag including your JS/TS script :

- In development mode, all scripts are included as modules.
- In development mode, all scripts are marked as `async` and `defer`.
- You can pass a second argument to this tag to overrides attributes
  passed to the script tag.
- This tag only accept JS/TS, for other type of assets, they must be
  included in the script itself using `import` statements.
- In production mode, the library will read the `manifest.json` file
  generated by ViteJS and import all CSS files dependent of this script
  (before importing the script).
- You can add as many of this tag as you want, for each input you specify
  in your ViteJS configuration file.
- The path must be relative to your `root` key inside your ViteJS config file.
- The path must be a key inside your manifest file `manifest.json` file
  generated by ViteJS.
- In general, this path does not require a `/` at the beginning
  (follow your `manifest.json` file).

```
{% vite_asset_url '<path to your asset>' %}
```

This will generate only the URL to an asset with no tag surrounding it.
**Warning, this does not generate URLs for dependant assets of this one
like the previous tag.**

```
{% vite_react_refresh %}
```
If you're using React, this will generate the Javascript `<script/>` needed to support React HMR.

```
{% vite_react_refresh nonce="{{ request.csp_nonce }}" %}
```

Any kwargs passed to vite_react_refresh will be added to its generated `<script/>` tag. For example, if your site is configured with a Content Security Policy using [django-csp](https://github.com/mozilla/django-csp) you'll want to add this value for `nonce`.

### Custom attributes

By default, all script tags are generated with a `type="module"` and `crossorigin=""` attributes just like ViteJS do by default if you are building a single-page app.
You can override this behavior by adding or overriding this attributes like so :

```jinja-html
{% vite_asset '<path to your asset>' foo="bar" hello="world" data_turbo_track="reload" %}
```

This line will add `foo="bar"`, `hello="world"`, and `data-turbo-track="reload"` attributes.

You can also use context variables to fill attributes values :

```
{% vite_asset '<path to your asset>' foo=request.GET.bar %}
```

If you want to overrides default attributes just add them like new attributes :

```
{% vite_asset '<path to your asset>' crossorigin="anonymous" %}
```

Although it's recommended to keep the default `type="module"` attribute as ViteJS build scripts as ES6 modules.

### Jinja2 template backend

If you are using Django with the Jinja2 template backend then alternative versions of the templatetags are provided, via an Extension.

If you are using [`django-jinja` library](https://niwi.nz/django-jinja/latest/#_add_additional_extensions) then you can:

```python
from django_jinja.builtins import DEFAULT_EXTENSIONS

"OPTIONS": {
    "extensions": DEFAULT_EXTENSIONS + [
        # Your extensions here...
        "django_vite.templatetags.jinja.DjangoViteExtension"
    ]
}
```

Or for a [vanilla Django + Jinja2 setup](https://docs.djangoproject.com/fr/4.2/topics/templates/#django.template.backends.jinja2.Jinja2) you can do something like:

```python
from django.templatetags.static import static
from django.urls import reverse

from jinja2 import Environment
from django_vite.templatetags.jinja import DjangoViteExtension


def environment(**options):
    options.setdefault("extensions", []).append(DjangoViteExtension)
    env = Environment(**options)
    return env
```

Then usage in your templates is similar, but adjusted as is normal for Jinja2 rather than Django template language.

- Omit the `{% load django_vite %}` which is not needed.
- Replace tag syntax with function calls, e.g.
  - `{% vite_react_refresh %}` becomes:
    `{{ vite_react_refresh() }}`
  - `{% vite_asset '<path to your asset>' foo="bar" %}` becomes:
    `{{ vite_asset('<path to your asset>', foo="bar") }}`
    etc

## Vite Legacy Plugin

If you want to consider legacy browsers that don't support ES6 modules loading
you may use [@vitejs/plugin-legacy](https://github.com/vitejs/vite/tree/main/packages/plugin-legacy).
Django Vite supports this plugin. You must add stuff in complement of other script imports in the `<head>` tag.

Just before your `<body>` closing tag add this :

```
{% vite_legacy_polyfills %}
```

This tag will do nothing in development, but in production it will loads the polyfills
generated by ViteJS.

And so next to this tag you need to add another import to all the scripts you have
in the head but the 'legacy' version generated by ViteJS like so :

```
{% vite_legacy_asset '<path to your asset>' %}
```

Like the previous tag, this will do nothing in development but in production,
Django Vite will add a script tag with a `nomodule` attribute for legacy browsers.
The path to your asset must contain de pattern `-legacy` in the file name (ex : `main-legacy.js`).

This tag accepts overriding and adding custom attributes like the default `vite_asset` tag.

## Multi-app configuration

If you would like to use django-vite with multiple vite configurations you can specify them in your settings.

```python
DJANGO_VITE = {
  "default": {
    "dev_mode": True,
  },
  "external_app_1": {
    ...
  },
  "external_app_2": {
    ...
  }
}
```

Specify the app in each django-tag tag that you use in your templates. If no app is provided, it will default to using the "default" app.

```html
{% vite_asset '<path to your asset>' %}
{% vite_asset '<path to another asset>' app="external_app_1" %}
{% vite_asset '<path to a third asset>' app="external_app_2" %}
```

You can see an example project [here](https://github.com/Niicck/django-vite-multi-app-example).

## Configuration Variables

You can redefine these values for each app config in `DJANGO_VITE` in `settings.py`.

### dev_mode
- **Type**: `bool`
- **Default**: `False`
- **Legacy Key**: `DJANGO_VITE_DEV_MODE`

Indicates whether to serve assets via the ViteJS development server or from compiled production assets.

Read more: [Dev Mode](#dev-mode)

### dev_server_protocol
- **Type**: `str`
- **Default**: `"http"`
- **Legacy Key**: `DJANGO_VITE_DEV_SERVER_PROTOCOL`

The protocol used by the ViteJS webserver.

### dev_server_host
- **Type**: `str`
- **Default**: `"localhost"`
- **Legacy Key**: `DJANGO_VITE_DEV_SERVER_HOST`

The `server.host` in `vite.config.js` for the ViteJS development server.

### dev_server_port
- **Type**: `int`
- **Default**: `5173`
- **Legacy Key**: `DJANGO_VITE_DEV_SERVER_PORT`

The `server.port` in `vite.config.js` for the ViteJS development server.

### static_url_prefix
- **Type**: `str`
- **Default**: `""`
- **Legacy Key**: `DJANGO_VITE_STATIC_URL_PREFIX`

The directory prefix for static files built by ViteJS.

- Use it if you want to avoid conflicts with other static files in your project.
- It's used in both dev mode and production mode.
- For dev mode, you also need to add this prefix inside vite config's `base`.
- For production mode, you may need to add this to vite config's `build.outDir`.

Example:

```python
# settings.py
DJANGO_VITE_STATIC_URL_PREFIX = 'bundler'
STATICFILES_DIRS = (('bundler', '/srv/app/bundler/dist'),)
```

```javascript
// vite.config.js
export default defineConfig({
  base: '/static/bundler/',
  ...
})
```

### manifest_path
- **Type**: `str | Path`
- **Default**: `Path(settings.STATIC_ROOT) / static_url_prefix / "manifest.json"`
- **Legacy Key**: `DJANGO_VITE_MANIFEST_PATH`

The absolute path, including the filename, to the ViteJS manifest file located in `build.outDir`.

### legacy_polyfills_motif
- **Type**: `str`
- **Default**: `"legacy-polyfills"`
- **Legacy Key**: `DJANGO_VITE_LEGACY_POLYFILLS_MOTIF`

The motif used to identify assets for polyfills in the `manifest.json`. This is only applicable if you are using [@vitejs/plugin-legacy](https://github.com/vitejs/vite/tree/main/packages/plugin-legacy).

### ws_client_url
- **Type**: `str`
- **Default**: `"@vite/client"`
- **Legacy Key**: `DJANGO_VITE_WS_CLIENT_URL`

The path to the HMR (Hot Module Replacement) client used in the `vite_hmr_client` tag.

### react_refresh_url
- **Type**: `str`
- **Default**: `"@react-refresh""`
- **Legacy Key**: `DJANGO_VITE_REACT_REFRESH_URL`

If you're using React, this will generate the Javascript needed to support React HMR.

## Notes

- In production mode, all generated paths are prefixed with the `STATIC_URL`
  setting of Django.

### Whitenoise

If you are serving your static files with whitenoise, by default your files compiled by vite will not be considered immutable and a bad cache-control will be set. To fix this you will need to set a custom test like so:

```python
import re

# http://whitenoise.evans.io/en/stable/django.html#WHITENOISE_IMMUTABLE_FILE_TEST

def immutable_file_test(path, url):
    # Match vite (rollup)-generated hashes, à la, `some_file-CSliV9zW.js`
    return re.match(r"^.+[.-][0-9a-zA-Z_-]{8,12}\..+$", url)


WHITENOISE_IMMUTABLE_FILE_TEST = immutable_file_test
```

## Examples

For examples of how to setup the project in v3, please see [django-vite-examples](https://github.com/Niicck/django-vite-examples).

For another example that uses the module-level legacy settings, please see this [example project here](https://github.com/MrBin99/django-vite-example).

## Thanks

Thanks to [Evan You](https://github.com/yyx990803) for the ViteJS library.
