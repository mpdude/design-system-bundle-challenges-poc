# Proof of Concept for loading SCSS from NPM packages installed through local links

## Following transitive NPM dependencies

### Assumptions

* We have something (our "design system bundle") that looks like an NPM package – i. e. contains a `package.json` file declaring NPM package dependencies.
* This "design system bundle" is installed through other means (for example, Composer) because we need to make sure other dependencies (PHP) match and are followed.
* After the "design system bundle" has been fetched, the NPM dependencies declared by it (in the "design system bundle" version chosen by the other package manager) have to be followed.

### Solution

* Use Yarn v2 and `protal:` link types
* https://yarnpkg.com/getting-started/migration describes how to get there
* Should work out of the box for anyone (i. e. no need to know up-front whether it's Yarn v1 or v2?)

### Example

[`packages/foo`](packages/foo) in this repo takes the role of such a "design system bundle" package. It does not matter if the directory has been created by some other package manager or is part of this repo.

`package.json` defines the `portal:` link for this package.

After `yarn install`, `node_modules/foo` is a symlink to `packages/foo`, and `packages/foo/node_modules` will contain `svala` v0.1.0 (as declared by the `foo` package).

Since the main `package.json` file declared `svala` v0.2.0 as a dependency, no hoisting for `svala` could be performed. The top-level `node_modules/svala` directory contains `svala` v0.2.0.

## Using `@import` in SCSS to refer to NPM packages

### Assumptions

* SCSS files need to be able to reference SCSS partials that are shipped as part of NPM packages
* When an SCSS file in the example `packages/foo` subpackage (our "design system bundle") `@imports` from `svala/...`, it should get the version declared in its own `package.json`. 
* An SCSS file in the "main" project (like `scss/main.scss`) should be able to ...
  * `@import` from the `foo` package
  * `@import` from `svala`, but get its own declared `svala` version then
* In case the main project and a subpackage end up using the same version of dependencies and hoisting is performed by Yarn, SASS path resolution must work as NPM's `require()` semantics dictate.

### Solution

* Use a custom `node-sass` [Importer](https://github.com/sass/node-sass#importer--v200---experimental). Importers can provide custom behaviour for `@import` SASS statements. SASS will first try to resolve the given import URL relative to the file containing the `@import` statement. After that, the Importer(s) will be run. If those are not successful, all SASS include paths will be tried as starting points.
* https://github.com/anarh/node-sass-import implements Node.js `require()`-style lookup behaviour for SASS
  * https://github.com/maoberlehner/node-sass-magic-importer looks promising as well with rich additional feature set, but: https://github.com/maoberlehner/node-sass-magic-importer/issues/188 – it does not honor the location of the file currently processed by SASS, thus not doing Node.js-style "searches" for `node_modules`, but effectively using only `node_modules` in the `cwd`.
* Write `@import "package/..."` in SASS. If `package` is not a subdirectory right next to the file being processed, it will be treated as an NPM package name.
* For the time being, https://github.com/webfactory/webfactory-gulp-preset/pull/14 must be used to set up the importer
* Having `~` as an additional qualifier might make this more explicit/avoid confusion?

### Example

Run `gulp css`. It will compile the CSS without failures.

[`scss/main.scss`](scss/main.scss) resolved `@foo/...` as `node_modules/foo/...`. 

In [`packages/foo/scss/subdir/screen.scss`](packages/foo/scss/subdir/screen.scss), `@import "svala/scss/tools/map-iterator"` was resolved to the `node_modules/svala` folder in `packages/foo`.

Now, change the toplevel `package.json` to require `svala: v0.1.0`, like the one in `package/foo`. Run `yarn install`. Now since both the main project and the "design system bundle" require the same version, Yarn will hoist the package. `gulp css` still works, as the SASS importer will now find `svala` in the top-level `node_modules` folder.

## Dealing with conflicting NPM dependencies ("there can only be one")

### Assumptions

* In some cases – probably like the SCSS imports demonstrated previously – the language (SCSS/CSS) may be able to deal with different versions of a package being installed. As long as SASS mixins or partials boil down to "local" CSS, it might be feasible to mix different versions of a package providing SASS files in a single resulting CSS file.
* This won't work for JavaScript/ECMAScript dependencies shared by the project and the "design system bundle", at least unless sophisticated bundling/scoping/... can be applied.
  * E. g. when the main project requires jQuery 3, the "design system bundle" however declares a jQuery 1 dependency, it's no use to get both versions installed at different locations in the `node_modules` tree
  * As of writing, we're not able to bundle our scripts in a way that would provide the right jQuery version to each part
  * For performance reasons (in the jQuery example), we'd rather see things fail fast instead of mixing installations.

### Solution

* It's a core design feature for Yarn/NPM to deal with "concurrent" package versions, since that is possible in the language (prototype-based inheritance in ECMAScript) – opposed to e. g. Composer/PHP.
* Use `peerDependency` declarations in the "design system bundle"
  * Will _not_ install a copy/alternative version in the sub-package
  * Will print a warning for missing/unmatched `peerDependency` (in Yarn v2, it's printed on every `yarn install` and `yarn explain peer-requirements ...` can be used to diagnose
  * `yarn check` has been removed in v2, but maybe the build could be failed otherwise

### Example

A warning is printed since the main `package.json` requires `jquery ^1.0`, but `packages/foo` has a `jquery ^3.0` `peerDependency`. Only `jquery` v1 is installed in `node_modules/jquery`.

Remove `jquery` from the main `package.json`. Now `yarn install` will say that the peer dependency is not provided (instead of being a wrong version).

Update the main `package.json` to use `jquery ^3.0`. The warning goes away when running `yarn install`.
