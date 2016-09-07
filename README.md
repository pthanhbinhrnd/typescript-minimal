Gulp

This quick start guide will teach you how to build TypeScript with gulp and then add Browserify, uglify, or Watchify to the gulp pipeline.

We assume that you’re already using Node.js with npm.
Minimal project

Let’s start out with a new directory. We’ll name it proj for now, but you can change it to whatever you want.

mkdir proj
cd proj

To start, we’re going to structure our project in the following way:

proj/
    +- src/
    +- dist/

TypeScript files will start out in your src folder, run through the TypeScript compiler and end up in dist.

Let’s scaffold this out:

mkdir src
mkdir dist

Initialize the project

Now we’ll turn this folder into an npm package.

npm init

You’ll be given a series of prompts. You can use the defaults except for your entry point. For your entry point, use ./dist/main.js. You can always go back and change these in the package.json file that’s been generated for you.
Install our dependencies

Now we can use npm install to install packages. First install TypeScript and gulp globally. (You might need to start npm install commands in this guide with sudo if you’re on a Unix system.)

npm install -g typescript gulp-cli

Then install gulp and gulp-typescript in your project’s dev dependencies. Gulp-typescript is a gulp plugin for Typescript.

npm install --save-dev gulp gulp-typescript

Write a simple example

Let’s write a Hello World program. In src, create the file main.ts:

function hello(compiler: string) {
    console.log(`Hello from ${compiler}`);
}
hello("TypeScript");

In the project root, proj, create the file tsconfig.json:

{
    "files": [
        "src/main.ts"
    ],
    "compilerOptions": {
        "noImplicitAny": true,
        "target": "es5"
    }
}

Create a gulpfile.js

In the project root, create the file gulpfile.js:

var gulp = require("gulp");
var ts = require("gulp-typescript");
var tsProject = ts.createProject("tsconfig.json");

gulp.task("default", function () {
    return tsProject.src()
        .pipe(ts(tsProject))
        .js.pipe(gulp.dest("dist"));
});

Test the resulting app

gulp
node dist/main.js

The program should print “Hello from TypeScript!”.
Add modules to the code

Before we get to Browserify, let’s build our code out and add modules to the mix. This is the structure you’re more likely to use for a real app.

Create a file called src/greet.ts:

export function sayHello(name: string) {
    return `Hello from ${name}`;
}

Now change the code in src/main.ts to import sayHello from greet.ts:

import { sayHello } from "./greet";

console.log(sayHello("TypeScript"));

Finally, add src/greet.ts to tsconfig.json:

{
    "files": [
        "src/main.ts",
        "src/greet.ts"
    ],
    "compilerOptions": {
        "noImplicitAny": true,
        "target": "es5"
    }
}

Make sure that the modules work by running gulp and then testing in Node:

gulp
node dist/main.js

Notice that even though we used ES2015 module syntax, TypeScript emitted CommonJS modules that Node uses. We’ll stick with CommonJS for this tutorial, but you could set module in the options object to change this.
Browserify

Now let’s move this project from Node to the browser. To do this, we’d like to bundle all our modules into one JavaScript file. Fortunately, that’s exactly what Browserify does. Even better, it lets us use the CommonJS module system used by Node, which is the default TypeScript emit. That means our TypeScript and Node setup will transfer to the browser basically unchanged.

First, install browserify, tsify, and vinyl-source-stream. tsify is a Browserify plugin that, like gulp-typescript, gives access to the TypeScript compiler. vinyl-source-stream lets us adapt the file output of Browserify back into a format that gulp understands called vinyl.

npm install --save-dev browserify tsify vinyl-source-stream

Create a page

Create a file in src named index.html:

<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8" />
        <title>Hello World!</title>
    </head>
    <body>
        <p id="greeting">Loading ...</p>
        <script src="bundle.js"></script>
    </body>
</html>

Now change main.ts to update the page:

import { sayHello } from "./greet";

function showHello(divName: string, name: string) {
    const elt = document.getElementById(divName);
    elt.innerText = sayHello(name);
}

showHello("greeting", "TypeScript");

Calling showHello calls sayHello to change the paragraph’s text. Now change your gulpfile to the following:

var gulp = require("gulp");
var browserify = require("browserify");
var source = require('vinyl-source-stream');
var tsify = require("tsify");
var paths = {
    pages: ['src/*.html']
};

gulp.task("copy-html", function () {
    return gulp.src(paths.pages)
        .pipe(gulp.dest("dist"));
});

gulp.task("default", ["copy-html"], function () {
    return browserify({
        basedir: '.',
        debug: true,
        entries: ['src/main.ts'],
        cache: {},
        packageCache: {}
    })
    .plugin(tsify)
    .bundle()
    .pipe(source('bundle.js'))
    .pipe(gulp.dest("dist"));
});

This adds the copy-html task and adds it as a dependency of default. That means any time default is run, copy-html has to run first. We’ve also changed default to call Browserify with the tsify plugin instead of gulp-typescript. Conveniently, they both allow us to pass the same options object to the TypeScript compiler.

After calling bundle we use source (our alias for vinyl-source-stream) to name our output bundle bundle.js.

Test the page by running gulp and then opening dist/index.html in a browser. You should see “Hello from TypeScript” on the page.

Notice that we specified debug: true to Browserify. This causes tsify to emit source maps inside the bundled JavaScript file. Source maps let you debug your original TypeScript code in the browser instead of the bundled JavaScript. You can test that source maps are working by opening the debugger for your browser and putting a breakpoint inside main.ts. When you refresh the page the breakpoint should pause the page and let you debug greet.ts.
Watchify, Babel, and Uglify

Now that we are bundling our code with Browserify and tsify, we can add various features to our build with browserify plugins.

    Watchify starts gulp and keeps it running, incrementally compiling whenever you save a file. This lets you keep an edit-save-refresh cycle going in the browser.

    Babel is a hugely flexible compiler that converts ES2015 and beyond into ES5 and ES3. This lets you add extensive and customized transformations that TypeScript doesn’t support.

    Uglify compacts your code so that it takes less time to download.

Watchify

We’ll start with Watchify to provide background compilation:

npm install --save-dev watchify gulp-util

Now change your gulpfile to the following:

var gulp = require("gulp");
var browserify = require("browserify");
var source = require('vinyl-source-stream');
var watchify = require("watchify");
var tsify = require("tsify");
var gutil = require("gulp-util");
var paths = {
    pages: ['src/*.html']
};

var watchedBrowserify = watchify(browserify({
    basedir: '.',
    debug: true,
    entries: ['src/main.ts'],
    cache: {},
    packageCache: {}
}).plugin(tsify));

gulp.task("copy-html", function () {
    return gulp.src(paths.pages)
        .pipe(gulp.dest("dist"));
});

function bundle() {
    return watchedBrowserify
        .bundle()
        .pipe(source('bundle.js'))
        .pipe(gulp.dest("dist"));
}

gulp.task("default", ["copy-html"], bundle);
watchedBrowserify.on("update", bundle);
watchedBrowserify.on("log", gutil.log);

There are basically three changes here, but they require you to refactor your code a bit.

    We wrapped our browserify instance in a call to watchify, and then held on to the result.
    We called watchedBrowserify.on("update", bundle); so that Browserify will run the bundle function every time one of your TypeScript files changes.
    We called watchedBrowserify.on("log", gutil.log); to log to the console.

Together (1) and (2) mean that we have to move our call to browserify out of the default task. And we have to give the function for default a name since both Watchify and Gulp need to call it. Adding logging with (3) is optional but very useful for debugging your setup.

Now when you run Gulp, it should start and stay running. Try changing the code for showHello in main.ts and saving it. You should see output that looks like this:

proj$ gulp
[10:34:20] Using gulpfile ~/src/proj/gulpfile.js
[10:34:20] Starting 'copy-html'...
[10:34:20] Finished 'copy-html' after 26 ms
[10:34:20] Starting 'default'...
[10:34:21] 2824 bytes written (0.13 seconds)
[10:34:21] Finished 'default' after 1.36 s
[10:35:22] 2261 bytes written (0.02 seconds)
[10:35:24] 2808 bytes written (0.05 seconds)

Uglify

First install Uglify. Since the point of Uglify is to mangle your code, we also need to install vinyl-buffer and gulp-sourcemaps to keep sourcemaps working.

npm install --save-dev gulp-uglify vinyl-buffer gulp-sourcemaps

Now change your gulpfile to the following:

var gulp = require("gulp");
var browserify = require("browserify");
var source = require('vinyl-source-stream');
var tsify = require("tsify");
var uglify = require('gulp-uglify');
var sourcemaps = require('gulp-sourcemaps');
var buffer = require('vinyl-buffer');
var paths = {
    pages: ['src/*.html']
};

gulp.task("copy-html", function () {
    return gulp.src(paths.pages)
        .pipe(gulp.dest("dist"));
});

gulp.task("default", ["copy-html"], function () {
    return browserify({
        basedir: '.',
        debug: true,
        entries: ['src/main.ts'],
        cache: {},
        packageCache: {}
    })
    .plugin(tsify)
    .bundle()
    .pipe(source('bundle.js'))
    .pipe(buffer())
    .pipe(sourcemaps.init({loadMaps: true}))
    .pipe(uglify())
    .pipe(sourcemaps.write('./'))
    .pipe(gulp.dest("dist"));
});

Notice that uglify itself has just one call — the calls to buffer and sourcemaps exist to make sure sourcemaps keep working. These calls give us a separate sourcemap file instead of using inline sourcemaps like before. Now you can run Gulp and check that bundle.js does get minified into an unreadable mess:

gulp
cat dist/bundle.js

Babel

First install Babelify. Like Uglify, Babelify mangles code, so we’ll need vinyl-buffer and gulp-sourcemaps.

npm install --save-dev babelify vinyl-buffer gulp-sourcemaps

Now change your gulpfile to the following:

var gulp = require('gulp');
var browserify = require('browserify');
var source = require('vinyl-source-stream');
var tsify = require('tsify');
var sourcemaps = require('gulp-sourcemaps');
var buffer = require('vinyl-buffer');
var paths = {
    pages: ['src/*.html']
};

gulp.task('copyHtml', function () {
    return gulp.src(paths.pages)
        .pipe(gulp.dest('dist'));
});

gulp.task('default', ['copyHtml'], function () {
    return browserify({
        basedir: '.',
        debug: true,
        entries: ['src/main.ts'],
        cache: {},
        packageCache: {}
    })
    .plugin(tsify)
    .transform("babelify")
    .bundle()
    .pipe(source('bundle.js'))
    .pipe(buffer())
    .pipe(sourcemaps.init({loadMaps: true}))
    .pipe(sourcemaps.write('./'))
    .pipe(gulp.dest('dist'));
});

We also need to have TypeScript target ES2015. Babel will then produce ES5 from the ES2015 code that TypeScript emits. Let’s modify tsconfig.json:

{
    "files": [
        "src/main.ts"
    ],
    "compilerOptions": {
        "noImplicitAny": true,
        "target": "es2015"
    }
}

Babel’s ES5 output should be very similar to TypeScript’s output for such a simple script.
