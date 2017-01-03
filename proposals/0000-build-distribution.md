- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)

# Summary

This RFC describes the proposed model for distributing Elix web components based on a monorepo repository structure. The document illustrates the key aspects of the monorepo file folder hierarchy, pertinent package.json files, how Elix is registered with npm, what an installed Elix distribution file folder tree looks like in an installed client's node_module folder, and how Elix elements are referenced in client web applications.

# Motivation

Elix is a library of web components based on a shared code base of mixin classes. The Elix project suggests that a common heritage of web components - sharing UI/UX standards, look and feel, and Gold Standard quality levels - will become a broader dependency for web application clients than individually selected components chosen from a wide range of providers. Elix is in part a statement about web component ecosystems, emphasizing the _collection_ of components over the cherry-picking selection of individual components.  

When you accept the premise of Elix as a library, along with the development efficiencies of organizing a project as a monorepo, then it becomes clear that distribution becomes a matter of installing the full Elix library on an application server, rather than individually installing separate components. This is especially true given the mixin-based architecture of Elix, where the bulk of the code is implemented in shared mixins rather than in the individual components themselves.  

Elix's distribution strategy is an efficient and opinionated model for building, distributing, installing, and accessing web components. The kernel of this opinionated model boils down to this: Only one package is registered with npm, and that is the full Elix package itself.

# Use cases

This RFC covers, from a client web application's perspective, how Elix components are implicitly registered through the Elix package with npm, and how they are installed and accessed under an application's node_module folder.

# Detailed design

In this and the [Alternatives](#alternatives) sections, we'll illustrate what the proposed git repository folder structure looks like, along with the repository's root package.json file. We'll also show how a client application using Elix lists its dependencies in its own package.json, and the resulting folder structure under its installed node_modules directory. Finally, we will show a simple client application index.html file example, indicating how the Elix JavaScript dependencies are referenced through script tags.

## Single Elix registered package

Under this distribution model, we look at the Elix repository as a unified JavaScript library of web components and associated, supporting mixin classes. A dependent application will use npm or Yarn to install the full library to its server, and reference web components from within that installed folder structure. An application under this model has a single dependency: the Elix package which itself has been registered with npm.

###Repository structure

This snapshot of Elix's git repository structure shows the location of the project's root _package.json_, and the organization of the _elements_ folder. In this example, there are two elements, _elix-element-1_ and _elix_element-2_, with the contents of _elix-element-1_ expanded for illustrative purposes. The _elix-mixins_ folder is a peer of the two element folders. Note also the location of the root _package.json_ file.  

Also notice that there are no _package.json_ files under the _elements_ folder or subfolders, indicating that nothing within the _elements_ folder tree gets registered individually with npm.

    /   
      /elements
        /elix-element-1
          /dist
            /elix-element-1.js
          /src
            /Element1.js
          /test
            /Element1.tests.js
          /globals.js
          /index.html
          /README.md
        /elix-element-2
        /elix-mixins 
      /package.json

### Project root package.json:

Elix's root _package.json_ file is what is registered with npm. Note that the repository URL points precisely to the root of the Elix git repository. A _.npmignore_ file within the root of the Elix repository will limit the files installed via npm or yarn, eliminating unnecessary repository content such as tests and build scripts.

    {   
      "name": "elix",
      "version": “1.0.0",
      "description": "Elix",
      "license": "MIT",
      "repository": {
        "type": "git",
        "url": "git://github.com/elix/elix.git"
      },
      "devDependencies": {
        ...
      },
      "browserify": {
        "transform": [
          "babelify"
        ]
      }
    }

### Application package.json

This is an example of the dependencies section of an Elix client application's _package.json_ file. The only Elix dependency is the single package described above, registered with npm and installed via npm or Yarn.

    {
      dependencies: {
        “elix”: “^1.0.0”,
        “other-app-dependency”: “^3.2.1”
      }
    }

### Distribution structure

Upon invoking an npm or Yarn install of Elix, the client application's _node_modules_ folder will look like the following. This is mostly a reflection of the Elix git repository's structure, filtered though a _.npmignore_ list of file exclusion directives.

    /node_modules
      /elix
        /elements
          /elix-element-1
            /dist
              /elix-element-1.js
            /src
              /Element1.js
            /test
              /Element1.tests.js
            /globals.js
            /index.html
            /README.md
          /elix-element-2
          /elix-mixins
        /package.json
      /other-app-dependency

### Client index.html

Based on the npm or Yarn installation of Elix, a client application will reference definitions for desired Elix elements in the following manner. The primary example shows reference to ES5-transpiled files, with examples for referencing ES2015 definitions immediately following.

    <!DOCTYPE html>
    <html lang="en">
    <head>
      <script src=“./node_modules/elix/elements/ElixElement1/dist/elix-element-1.js">
      </script>
      <script src=“./node_modules/elix/elements/ElixElement2/dist/elix-element-2.js”>
      </script>
    </head>
    <body>
      <elix-element-1>Elix</elix-element-1>
      <elix-element-2>More Elix</elix-element-2>
    </body>
    </html>

#### ES2015 script references

    <script src=“./node_modules/elix/elements/ElixElement1/src/ElixElement1.js”></script>
    <script src=“./node_modules/elix/elements/ElixElement2/src/ElixElement2.js”></script>
    
# Drawbacks

Drawbacks include:

1. Search within npm for an elix-* component name may be difficult or impossible.
2. There is a common developer perspective that individual web components are to be registered with npm rather than collections of components. This approach is a relatively new and opinionated model.
3. Versioning is applied to the entire collection of web components. While this may be exactly the right way to version collections of components, developers may need to be educated as to the underlying (mixin-based) architecture of the component collection in order to understand the reasons for the common versioning model.

# Alternatives

We illustrate two alternatives to the single Elix package distribution model:

1. The same as the proposed model above, but register Elix elements individually with npm
2. Separately register and distribute Elix elements and the Elix mixin package

## Proposed model with npm registration of individual elements

This model varies from the proposed model only in that _package.json_ files are specified for each Elix element under the _elements_ folder. Each _element_ folder contains a _package_ folder that contains a _package.json_ file for the particular element. By adding a _package_ folder, we can register a minimal source tree -- essentially only the element's _package.json_ file -- with npm for each element. The _package.json_ file simply points to the Elix unified package as its dependency.

All this model provides over the proposed model is the ability to search for specific Elix elements in the npm registry. The same unified Elix package is installed once for any client application using this distribution model.

###Repository structure

Note the sole difference with the proposed model: the addition of a _package_ folder under each _element_ folder. The repository's root _package.json_ remains as before.

    /
      /elements
        /elix-element-1
          /dist
            /elix-element-1.js
          /package
            /package.json
          /src
            /Element1.js
          /test
            /Element1.tests.js
          /globals.js
          /index.html
          /README.md
        /elix-element-2
        /elix-mixins
    /package.json

### Element package.json

This is an example of one element's _package.json" files, which gets registered with npm under the element's name. Note in particular that it has a single dependency -- the unified Elix package -- and its repository key points to the _package_ folder for the element within the Elix respository tree.

    {
      "name": “elix-element-1",
      "version": “1.0.0",
      "description": “Elix Element 1",
      "license": "MIT",
      "repository": "https://github.com/elix/elix/tree/master/elements/elix-element-1/package”,
      “dependencies”: {
        “elix”: “1.0.0”
      }
    }

### Application package.json

A client application's _package.json_ file includes the specific Elix elements it wants. As each Elix element package has a dependency on the unified Elix package, that package does not need to be specified (or even understood) by the client application developer.

    {
      dependencies: {
        “elix-element-1”: “^1.0.0”,
        “elix-element-2”: “^1.0.0”,
        “other-app-dependency”: “^3.2.1”
      }
    }

### Distribution structure

With an npm or Yarn installation, the _node_modules_ folder structure is the same as the proposed model with the addition of the individual Elix elements as peer folders to the installed _elix_ folder. Remember that the _elix_ folder is installed as a result of it being a dependency of all Elix element packages.  

The installed element folders are mostly empty, containing just the element's _package.json_ and possibly a redirection JavaScript file for developer convenience that points to the element's definition within the _elix_ folder tree.

    /node_modules
      /elix
        /elements
          /elix-element-1
            /dist
              /elix-element-1.js
            /src
              /Element1.js
            /test
              /Element1.tests.js
            /globals.js
            /index.html
            /README.md
          /elix-element-2
          /elix-mixins
        /package.json
      /elix-element-1
        /package.json
      /elix-element-2
        /package.json
      /other-app-dependency

### Client index.html

Without the developer convenience of an element redirection JavaScript file, the client application's HTML will look the same as the proposed model.

    <!DOCTYPE html>
    <html lang="en">
    <head>
      <script src=“./node_modules/elix/elements/ElixElement1/dist/elix-element-1.js">
      </script>
      <script src=“./node_modules/elix/elements/ElixElement2/dist/elix-element-2.js”>
      </script>
    </head>
    <body>
      <elix-element-1>Elix</elix-element-1>
      <elix-element-2>More Elix</elix-element-2>
    </body>
    </html>

#### ES2015 script references

    <script src=“./node_modules/elix/elements/ElixElement1/src/ElixElement1.js”></script>
    <script src=“./node_modules/elix/elements/ElixElement2/src/ElixElement2.js”></script>

## Register individual elements and the elix-mixin package
### Repository structure

    /
      /elements
        /elix-element-1
          /dist
            /elix-element-1.js
          /package
            /package.json
          /src
            /Element1.js
          /test
            /Element1.tests.js
          /globals.js
          /index.html
          /README.md
        /elix-element-2
        /elix-mixins
    /package.json

### Elix-mixins package.json

    {
      "name": “elix-mixins",
      "version": “1.0.0",
      "description": “Elix Mixins",
      "license": "MIT",
      "repository": "https://github.com/elix/elix/tree/master/elements/elix-mixins/package”,
      “dependencies”: {
      }
    }

### Element package.json

    {
      "name": “elix-element-1",
      "version": “1.0.0",
      "description": “Elix Element 1",
      "license": "MIT",
      "repository": "https://github.com/elix/elix/tree/master/elements/elix-element-1/package”,
      “dependencies”: {
        “elix-mixins”: “1.0.0”
      }
    }

### Application package.json

    {
      dependencies: {
        “elix-element-1”: “^1.0.0”,
        “elix-element-2”: “^1.0.0”,
        “other-app-dependency”: “^3.2.1”
      }
    }

### Distribution structure

    /node_modules
      /elix-element-1
        /dist
          /elix-element-1.js
        /src
          /Element1.js
        /globals.js
        /index.html
        /README.md
      /elix-element-2
      /elix-mixins
      /other-app-dependency

### Client index.html

    <!DOCTYPE html>
    <html lang="en">
    <head>
      <script src=“./node_modules/elix-mixins/dist/elix-mixin-1.js”></script>
      <script src=“./node_modules/elix-mixins/dist/elix-mixin-2.js”></script>
      <script src=“./node_modules/elix-element-1/dist/elix-element-1.js"></script>
      <script src=“./node_modules/elix-element-2/dist/elix-element-2.js”></script>
    </head>
    <body>
      <elix-element-1>Elix</elix-element-1>
      <elix-element-2>More Elix</elix-element-2>
    </body>
    </html>

#### ES2015 script references

    <script src=“./node_modules/elix-mixins/src/ElixMixin1.js
    <script src=“./node_modules/elix-mixins/src/ElixMixin2.js
    <script src=“./node_modules/elix-element-1/src/ElixElement1.js”></script>
    <script src=“./node_modules/elix-element-2/src/ElixElement2.js”></script>

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?