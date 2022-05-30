# Geotab Developers

> This project serves as the online SDK for the MyGeotab, Geotab Drive, and MyAdmin. It also contains the guidelines and protocols to integrate IOX hardware add-ons with the GO device as well as Custom Telematics Devices using Geotab’s Data Intake Gateway (DIG). Everything is hosted via Github pages.

## Getting Started

Make sure you have installed:

* **Ruby**: `>2.0` with Bundler `>1.10`
* **Node**: `>17.3.1`
* **Gulp:** `4.0` : `npm install gulp-cli -g`

If you have cloned this repo or want to reinstall, make sure there's no `node_modules` or `Gemfile.lock` folder/file and then run:

```sh
$ npm install
$ bundle install
```

For local testing run with command:

```sh
$ gulp
```

## Deployment

The documention can be built and deployed using these commands:

```sh
$ gulp build --prod
```

```sh
$ gulp deploy
```

### Settings

The `_config.yml` and `_config.build.yml` provide the Jekyll settings. `_config.build.yml` is only used in production build and overrides settings in `_config.yml`;

## Dependencies

> generator-jekyllized
For more information on how to use this project, please refer to the [README
on Github](https://github.com/sondr3/generator-jekyllized).

## Owner

> [Geotab Inc.](https://my.geotab.com)
