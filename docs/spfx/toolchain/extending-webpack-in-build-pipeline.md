---
title: Extending Webpack in the SharePoint Framework toolchain
ms.date: 09/25/2017
ms.prod: sharepoint
---


# Extending Webpack in the SharePoint Framework toolchain

[Webpack](https://Webpack.js.org/) is a JavaScript module bundler that takes your JavaScript files and their dependencies and generates one or more JavaScript files so you can load different pieces of code for different scenarios.

The framework toolchain uses CommonJS for bundling. This enables you to define modules and where you want to use them. The toolchain also uses SystemJS, a universal module loader, to load your modules. This helps you to scope your web parts by making sure that each web part is executed in its own namespace.

One common task you would want to add to the SharePoint Framework toolchain is to extend the Webpack configuration with custom loaders and plugins.

## Using Webpack loaders
There are many cases where one would like to import and utilize a non-JavaScript resource during development, typically this is done with images or templates. A [Webpack loader](https://webpack.js.org/loaders/) will convert the resource into something that can be utilized by your JavaScript application or provide a simple reference that your JavaScript application can understand. For example, a Markdown template may be compiled and converted to a text string, while a image resource may be converted to an inlined Base64 image, or it may be referenced as a URL and copied to your `dist` directory for deployment.

There are a number of useful loaders, several of which are already used by the standard SharePoint Framework Webpack configuration, such as:

- html-loader
- json-loader
- loader-load-themed-styles

Extending the framework Webpack configuration with custom loaders is a straightforward process which is documented [here in the Webpack documentation](https://webpack.js.org/contribute/writing-a-loader/).

> You can find more details on the loaders from [Webpack loaders documentation](https://webpack.js.org/loaders/)

## Example: Using the markdown-loader package
As an example, let's use the [markdown-loader package](https://www.npmjs.com/package/markdown-loader).  It's a loader which allows you to reference an `.md` file and output it as an HTML string.

You can download the completed sample [here](https://aka.ms/spfx-extend-Webpack-sample).

### Step 1 - Install the package
Let's reference markdown-loader in our project.

```
npm i --save markdown-loader
```

### Step 2 - Configure Webpack
Now that we have the package installed, lets now configure the SharePoint Framework Webpack configuration to include `markdown-loader`.

In the [documentation of markdown-loader](https://github.com/peerigon/markdown-loader), it shows how to extend the Webpack configuration to include the loader:

```JavaScript
{
  module: {
    rules: [
      {
        test: /\.md$/,
        use: [
          {
            loader: 'html-loader'
          },
          {
            loader: 'markdown-loader',
            options: {
              /* options for markdown-loader here */
            }
          }
        ]
      }
    ]
  }
}
```

Let's take a look at what this configuration is doing:
  - The `rules` array in the Webpack configuration defines a set of file path tests and the loaders that should be used when a resource is found that matches the test. In this case, the `test` property checks for file paths that end with `.md`.
  - The `use` array describes a list of loaders that are applied sequentially to the resource. They are applied from last to first. In this case, the first loader to be applied will be `markdown-loader` and the last loader to be applied will be `html-loader`.
  - When multiple loaders are specified, the result of each loader is piped to the next.

We will use this information to configure it in our project.

In order to add this custom loader into the SharePoint Framework Webpack configuration, we will need to instruct the build task to configure Webpack. The build tasks are defined in the gulp file - `gulpfile.js` - which is located at the root of your project.

Edit the `gulpfile.js` and add the following code right before `build.initialize(gulp);`:

```JavaScript
build.configureWebpack.mergeConfig({
  additionalConfiguration: (generatedConfiguration) => {
    generatedConfiguration.module.rules.push(
      {
        test: /\.md$/,
        use: [
          {
            loader: 'html-loader'
          },
          {
            loader: 'markdown-loader'
          }
        ]
      }
    );

    return generatedConfiguration;
  }
});
```

Let's walk through what this code snippet is doing:
  - As its name implies, the `ConfigureWebpackTask` (instantiated as `build.configureWebpack`) configures webpack for us. There is a lot of special configuration that happens to build SPFx projects, so there is some nontrivial logic in this task.
  - The `ConfigureWebpackTask` takes an optional `additionalConfiguration` property. We want to set this property to a function which takes the generated configuration, makes our modifications to it, and then returns the updated configuration. **Note that this function must return a Webpack configuration to the toolchain, otherwise Webpack will not be configured correctly.**
  - Inside the body of the function that we set to `additionalConfiguration`, simply push a new rule onto the existing set of rules in the configuration. Notice that this new rule is very similar to example in the configuration snippet at the top of **Step 2**.

> While you are able to completely replace the toolchain's default Webpack configuration using this approach, to get the maximum benefit with performance and optimization, it is not recommended to do so unless stated otherwise in the documentation.

### Step 3 - Update your code
Now that we have configured the loader, lets update our code and add few files to test the scenario.

Create a file `my-markdown.md` in the `src` directory of your project folder with some Markdown text in it.

```md
#Hello Markdown

*Markdown* is a simple markup format to write content.

You can also format text as **bold** or *italics* or ***bold italics***
```

When you build the project, the Webpack markdown-loader will convert this markdown text to a HTML string. To use this HTML string in any of your source `*.ts` files, add the following `require()` line at the top of the file after your imports, for example:


```TypeScript
const markdownString: string = require<string>('./../../../../src/readme.md');
```

Webpack by default will look in the `lib` folder for the file, but by default `.md` files don't get copied to the `lib` folder, meaning we need to create a rather lengthy relative path. We can control this setting by defining a config file to tell the toolchain to copy `md` files to the lib folder.

Create a file `copy-static-assets.json` in the `config` directory to tell the build system to copy some additional files from `src` to `lib`. By default, this build task copies files with extensions that the default toolchain Webpack configuration understands (like `png` and `json`), so we just need to tell it to also copy `md` files.

```JSON
{
  "includeExtensions": [
    "md"
  ]
}
```

Now instead of using the relative path, you can use the file path in your `require` statement, for example:

```TypeScript
const markdownString: string = require<string>('./../../readme.md');
```

You can then reference this string in your code, for example:

``` TypeScript
public render(): void {
  this.domElement.innerHTML = markdownString;
}
```

### Step 4 - Build and test your code
To build and test your code, execute the following command in a console in the root of your project directory:

```
gulp serve
```
