# Gulp setup for the MWS Phase 1 to 3 project
---

## What is it?

* Uses a `src` folder for source files, and a `dist` folder for distribution files. 
* Creates responsive images of different sizes and quality, for all `.jpg` files, using [gulp-responsive](https://www.npmjs.com/package/gulp-responsive).
* Creates javascript [bundles using Browserify with Watchify](https://github.com/gulpjs/gulp/blob/master/docs/recipes/fast-browserify-builds-with-watchify.md), so you can `require` or `import` scripts. 
* Copies all files not handled by any gulp task, into the `dist` folder.
* Allows you to add more tasks if you wish to handle other stuff, like sass/scss to css, etc.

## How to use.

### The `src`, `src/js`, `src/css` and `src/images` folders.

Place all your source files in these folders. This includes all files required to run your web application, like html, css, javascript, images, ect.
  * Place `index.html` and `restaurant.html` and your `sw.js` files in the root of the `src/` folder.
  * Place all your javascript files, **except** for `sw.js`, in the `src/js/` folder. These include `main.js`, `restaurant_info.js` and `dbhelper.js`. If you plan on adding more javascript files, like a javascript file containing your secret `MAPBOX API KEY`, you can do so here.
  * Place all your css files in `src/css`. 
  * Place all your images in `src/images/`. Once responsive images are created, for `.jpg` files only, they'll be placed on the `dist/img/` folder using the settings found in the `gulpfile`.
  * All other images, like `png` or `gif`, can be placed in the `src/images/` folder as well, but they will be copied into `dist/images/`, **not** `dist/img/`

### .gitkeep files

Once you add your source files to `src/`, `src/css/', `src/js/'` and `src/images/`, you can delete this files. They are there only to allow all this folders to be part of the repository. So as long as the folder isn't empty, git will keep this folders.

### Bundles

This setup creates three bundles: 

* `main.js`: bundle in `dist/js/main.js`
* `restaurant_info.js`: bundled in `dist/js/restaurant_info.js`
* `sw.js`: bundled in `dist/sw.js`

This will allow you to use `require` and `import` statements in any of these files, and also in files you import there. The reason three bundles are created, is because we have two `html` files, `index.html` and `restaurant.html`, and the service worker should have it's own file.

### Responsive images

This setup has a `responsive:images` task that creates 3 different size/quality images. You can modify the sizes, quality and number of images to create in this task.
```javascript
// gulpfile line 51-73
// task for creating responsive images
gulp.task('responsive:images', function() {
  log(c.cyan('Creating Responsive images...'));
  return gulp.src(paths.responsive.src)
    .pipe(responsive({
      // Here is where you can change sizes and suffixes to fit your needs. Right now 
      // we are resizing all jpg images to three different sizes: 300, 600 and 800 px wide.
      '**/*.jpg': [{
        width: 800,
        quality: 70,
        rename: { suffix: '-large'}
      }, {
        width: 600,
        quality: 50,
        rename: { suffix: '-medium'}
      }, {
        width: 300,
        quality: 40,
        rename: { suffix: '-small'}
      }]
    },))
    .pipe(gulp.dest(paths.responsive.dest));
});
```

### Changing the port number

To change the port number BrowserSync uses, change the port number in the `gulp sync` task.
```javascript
// gulpfile startin on line 93
// Browser sync task to use in development.
gulp.task('sync', ['build'], function() {
  browserSync.init({
    port: 8000,
    server: {
      baseDir: './dist'
    }
  });
  //...
```

### The `gulpfile` and the `gulp sync` task

The `gulpfile` has a `gulp sync` task that will create responsive images, create javascript bundles, and copy any other type of file into the `dist` folder. 
  * You can change the settings on the `responsive:images` task so the number of images, their sizes and quality match what you need.

## Adding gulp tasks

### The `paths` object in the `gulpfile`, and the `gulp copy` task.

You should add all the paths you'll handle in any new gulp tasks here. The use of this object is important, as the `gulp copy` task relies on this object to know what files are being handled by a task, and what files aren't so it should just make a copy of them into the `dist` folder.

``` javascript
// gulpfile line 19-31
// Add src and dest paths to files you will handle in tasks here. For js files, also add bundles to create
const paths = {
  responsive: {
    src: 'src/images/**/*.jpg',
    dest: 'dist/img/'
  },
  js: {
    src: 'src/**/*.js',
    dest: 'dist/',
    // don't add the src folder to path. Use a path relative to the src folder. Use array even if only one bundle.
    bundles: ['js/main.js', 'js/restaurant_info.js', 'sw.js']
  }
};
```

 For example, if you later decide to handle all `css` files, you'd add a new property to your paths object like this:
```javascript
const paths = {
  responsive: {
    //..
  },
  js: {
    //...
  },
  css: {
    src: 'src/css/**/*.css',
    dest: 'dist/css'
  }
};
```

Then you'll use `paths.css.src` as the source, and `paths.css.dest` as the destination in your gulp tasks:

```javascript
gulp.task('css', function() {
  return gulp.src(paths.css.src)
    .pipe(CSS_PLUGIN)
    .pipe(gulp.dest(paths.css.dest));
});
```

And now the `gulp copy` task will know it shouldn't try to copy files found in `paths.css.src`, since now a task is taking care of files found in this location.

### Add your tasks to the `gulp build` task

When you add tasks, add them to the `gulp build` task, so they are run in the right order, using `runSequence()`. Just place your tasks in the sequence you like (that's what `runSequence()` is for), or if it doesn't matter, just place them inside the array where `responsive:images` and `js:bundle` tasks are.

```javascript
// build task
gulp.task('build', function(done) {
  return runSequence(
    'clean',
    ['responsive:images', 'js:bundle'],
    'copy', // copy is done last, so is easy to see what's been copied.
    done
  )
});
```

### Watch the source of your new tasks in the `gulp sync` task

Since is very likely you want *Gulp* to watch for changes made in the source of your new tasks, just add a `gulp.watch` for every source you added to the `paths` object, and have it run the required task, and then on change, have it reload BrowserSync. 

You can use the `gulp.watch` used for `responsive:images` as an example. Or if your task requires special attention, like the `js:bundle` task, you'll have to implement it yourself, and use `browserSync.stream()`.

```javascript
// gulpfile line 93-116
// Browser sync task to use in development.
gulp.task('sync', ['build'], function() {
  //..
  
  // Add more watchers here if you add more tasks. Use this watcher as an example
  gulp.watch(paths.responsive.src, ['responsive:images']).on('change', browserSync.reload);

  // if your task requires special attention, then implement a different way of watching for
  // changes, and use browserSync.stream(). For example, here each bundle on 'update' will
  // call browserSync.stream() at the end of the pipe in the bundle() function.
  Object.keys(jsBundles).forEach(function(key) {
    var b = jsBundles[key];
    b.on('update', function() {
      bundle(b, key); // do not use return, or else only one bundle will be created
    });
  });
});
```

### Adding or removing Javascript bundles

Say you want one bundle for your `html` files and one for your service worker, or perhaps, you want to add more bundles instead. The `js:bundle` task uses the `paths.js.bundles` array to create bundles, so just list your desired bundles there.

``` javascript
// gulpfile line 19-31
// Add src and dest paths to files you will handle in tasks here. For js files, also add bundles to create
const paths = {
  //...
  js: {
    //...
    // don't add the src folder to path. Use a path relative to the src folder. Use array even if only one bundle.
    bundles: ['js/main.js', 'js/restaurant_info.js', 'sw.js']
  }
};
```

So if you wanted only 2 bundles, `main.js` and `sw.js`:

``` javascript
// gulpfile line 19-31
// Add src and dest paths to files you will handle in tasks here. For js files, also add bundles to create
const paths = {
  //...
  js: {
    //...
    // don't add the src folder to path. Use a path relative to the src folder. Use array even if only one bundle.
    bundles: ['js/main.js', 'sw.js']
  }
};
```

___
**Please star repo if you found gulp setup useful :)**
