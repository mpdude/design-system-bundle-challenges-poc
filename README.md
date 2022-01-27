# "Design System Bundle" POC

How can we install a PHP package through Composer, and then

* also fetch NPM dependencies declared by the version installed,
* refer to SCSS files _in_ that package
* refer to other SCSS files in NPM packages required _by_ that package
* avoid mixing incompatible library versions _across_ NPM packages
* reference assets (like images) from and across such NPM packages?

## Following transitive NPM dependencies for a "local" package

### Challenge

* We have something (our "design system bundle") that looks like an NPM package – i. e. contains a `package.json` file declaring NPM package dependencies. 
* This "local package" is installed through other means (for example, Composer) because we need to make sure other dependencies (PHP) match and are followed.
* After the local package has been installed through other means, the NPM dependencies declared by it (in the version checked out by the other package manager) have to be fetched as well.

### Problems 

* The `link:` protocol in Yarn v1 does not set up `node_modules` in that local package. Its purpose is [somewhat explained in the Yarn V2 docs](https://yarnpkg.com/features/protocols#whats-the-difference-between-link-and-portal). The [initial implementation](https://github.com/yarnpkg/yarn/pull/3359) is not very clear on the semantics of a `link:`, and [other people were confused as well](https://github.com/yarnpkg/yarn/issues/5341).
* At least in the past, there [were issues](https://github.com/yarnpkg/yarn/pull/2860) with `file:` references that would copy the local package into the cache but use the cache from then on, effectively never picking up local changes. Additionally, the copy performed by `file:` is slow, expensive and gets in the way when you need to make changes to the local package.

### Solution

* Use Yarn v2 and `protal:` link types
* https://yarnpkg.com/getting-started/migration describes how to get there
* Should work out of the box for anyone (i. e. no need to know up-front whether it's Yarn v1 or v2?)
* Local package will by symlinked into the top-level `node_modules` folder. It may have its own `node_modules` folder set up, depending on whether that is necessary or package hoisting will be performed.

### Example

[`packages/foo`](packages/foo) in this repo takes the role of such a "design system bundle" package. It does not matter if the directory has been created by some other package manager or is part of this repo.

`package.json` defines the `portal:` link for this package.

After `yarn install`, `node_modules/foo` is a symlink to `packages/foo`, and `packages/foo/node_modules` will contain `svala` v0.1.0 (as declared by the `foo` package).

Since the main `package.json` file declared `svala` v0.2.0 as a dependency, no hoisting for `svala` could be performed. The top-level `node_modules/svala` directory contains `svala` v0.2.0.

## Using `@import` in SCSS to refer to NPM packages

### Challenge

* SCSS files need to be able to reference SCSS partials that are shipped as part of NPM packages
* When an SCSS file in the example `packages/foo` subpackage (our "design system bundle") `@imports` from `svala/...`, it should get the version declared _in its own_ `package.json`. 
* An SCSS file in the "main" project (like `scss/main.scss`) should be able to ...
  * `@import` from the `foo` package
  * `@import` from `svala`, but also _get its own_ declared `svala` version then
* In case the main project and a subpackage end up using the same version of dependencies and hoisting is performed by Yarn, SASS path resolution must work as NPM's `require()` semantics dictate.

### Solution

* Use a custom `node-sass` [Importer](https://github.com/sass/node-sass#importer--v200---experimental). Importers can provide custom behaviour for `@import` SASS statements. 
* SASS will first try to resolve the given import URL relative to the file containing the `@import` statement. After that, the Importer(s) will be run. If those are not successful, all SASS `includePaths` will be tried as starting points.
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

### Challenge

* In some cases – like the SCSS imports demonstrated previously – the language (SCSS/CSS) may be able to deal with different versions of a package being installed. As long as SASS mixins or partials boil down to generating CSS snippets, it might be feasible to mix different versions of a package providing SASS files in a single resulting CSS file.
* This won't work for JavaScript/ECMAScript dependencies shared by the project and the other packages, at least unless sophisticated bundling/scoping/... can be applied.
  * E. g. when the main project requires jQuery 3, the "design system bundle" however declares a jQuery 1 dependency, it's no use to get both versions installed at different locations in the `node_modules` tree.
  * As of writing, we're not able to bundle our scripts in a way that would provide the right jQuery version to each part.
  * For performance reasons (in the jQuery example), we'd rather see things fail fast instead of mixing installations.

### Solution

* It's a core design feature for Yarn/NPM to deal with "concurrent" package versions, since that is possible in the language (prototype-based inheritance in ECMAScript) – opposed to e. g. Composer/PHP.
* Use `peerDependency` declarations in the "design system bundle":
  * Will _not_ install a copy/alternative version in the sub-package
  * Will print a warning for missing/unmatched `peerDependency` (in Yarn v2, it's printed on every `yarn install` and `yarn explain peer-requirements ...` can be used to diagnose
  * `yarn check` has been removed in v2, but maybe the build could be failed otherwise

### Example

A warning is printed since the main `package.json` requires `jquery ^1.0`, but `packages/foo` has a `jquery ^3.0` `peerDependency`. Only `jquery` v1 is installed in `node_modules/jquery`.

Remove `jquery` from the main `package.json`. Now `yarn install` will say that the peer dependency is not provided (instead of being a wrong version).

Update the main `package.json` to use `jquery ^3.0`. The warning goes away when running `yarn install`.

## Referencing assets from NPM packages in SASS/CSS `url()`s, rebasing

### Challenge

* In SCSS, you need to reference assets (like images, icons) that may be part of NPM packages.
* It would be beneficial if we could use `url(...)` paths _relative to the file that contains the statement_:
  * Can be understood/followed/auto-completed by IDEs without knowning how SASS files will be compiled and/or nested (imported) into each other
  * No need to define per-package prefixes externally
  * You should be able to reason about what an SCSS file does by looking at the file only, where it is located within its own package and maybe what dependencies are declared by that package.
* Problem: SCSS is imported/used across many levels. The `url()` statement is just a CSS rule that may end up in CSS file in completely different locations, but is not processed during SASS compilation.
* Due to that, we need to rebase URLs anyway. But: Currently, rebasing happens _after_ SCSS has been compiled down to CSS, not _for every SCSS file individually_.
* As with SCSS files themselves, if we want to reference assets from NPM packages, `require()` semnatics and/or hoisting rules need to be known
  * Assume two packages declare a dependency on `some-iconset` in `v1` and `v2`, respectively. Then the same `url()` in SASS files of either package needs to resolve to (possibly) different image files.
* URL rebasing needs to collect the referenced assets from the package and `node_modules` tree, which may not be publicly accessible at all. Targeted files need to be copied to a public directory. Possible filename clashes (previous example) need to be dealt with.

### Solution ideas

* Again, use the SASS Importer to find `url()` references whilst a SASS file is being loaded
* Rewrite all `url()` to be absolute file paths on disk; based on the current file path – temporary solution while CSS is being compiled
* For assets in `node_modules`, possible hoisting must be taken into account
* Clean up final CSS with something like [postcss-url](https://github.com/postcss/postcss-url), replacing filesystem-based absolute paths with paths to a public staging directory where assets are being collected
* Problem: SASS Importer not being used when imports can be resolved directly (e. g. `@import "some/partial"` will find `./some/_partial.scss` without invoking the  Importer).

### Solution ideas 2

* https://github.com/bholloway/resolve-url-loader/blob/HEAD/packages/resolve-url-loader/docs/how-it-works.md – This is for Webpack, but maybe the general idea still holds.
* After the final CSS has been generated, the sourcemap tells which original SCSS file contributed each character in the CSS. 
* Find `url()` references in the CSS
* Look up the originating SCSS file by means of the sourcemap
* Resolve the `url()` relative to that file
* When no match was found (and/or possibly the URL starts with a special `~packagename` identifier?), apply the `require()` loading mechanism to traverse the `node_modules` hierarchy; start at the SCSS file containing the URL.
* If necessary/helpful, change URLs to absolute, file-system based paths during processing. Use a final [postcss-url](https://github.com/postcss/postcss-url) step to gather files in a public `dist` area and change URL paths accordingly.

### SASS Rabbit Hole?

The [sass-maze](sass-maze) folder illustrates the challenges that may arise.

Compile `styles.scss` to `.css` by running `npx node-sass styles.scss --output . --output-style expanded --source-map true` and inspect the source map with a tool like http://sokra.github.io/source-map-visualization/#custom.

