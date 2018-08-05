<iframe 
  height='265' 
  scrolling='no' 
  title='Day 8: Metaballs' 
  src='//codepen.io/Uberche/embed/yqVgqE/?height=265&theme-id=0&default-tab=css,result&embed-version=2' 
  frameborder='no' 
  allowtransparency='true' 
  allowfullscreen='true' 
  style='width: 100%;'>
  
  See the Pen <a href='https://codepen.io/Uberche/pen/yqVgqE/'>Day 8: Metaballs</a> by Ethan (<a href='https://codepen.io/Uberche'>@Uberche</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>

See the Pen <a href='https://codepen.io/Uberche/pen/yqVgqE/'>Day 8: Metaballs</a> by Ethan (<a href='https://codepen.io/Uberche'>@Uberche</a>) on <a href='https://codepen.io'>CodePen</a>.

See the Pen [Day 8: Metaballs](1) by Ethan ([@Uberche](2)) on [CodePen](3).

![][image-1]

It’s always important to measure and then optimize. Lighthouse’s load performance audits look at:

*   [**First meaningful paint**](4) (when is the main content of the page visible)
*   [**Speed Index**](5) (visual completeness)
*   **Estimated Input Latency** (when is the main thread available to immediately handle user input)
*   and **Time To Interactive** (how soon is the app usable & engagable)

_Btw, Paul Irish has done terrific work summarising_ [_interesting metrics for PWAs_](6) _worth keeping an eye out._

**Good performance goals:**

*   **Follow L in the** [**RAIL performance model**](7)**.** _A+ performance is something all of us should be striving for even if a browser doesn’t support Service Worker. We can still get something meaningful on the screen quickly & only load what we need_
*   **Under representative network (3G) & hardware conditions**
*   Be interactive in < 5s on first visit & < 2s on repeat visits once a Service Worker is active.
*   First load (network-bound), Speed Index of 3,000 or less
*   Second load (disk-bound because SW): Speed Index of 1,000 or less.

Let’s talk a little more about focusing on interactivity via TTI.

Optimizing for interactivity means making the app usable for users as soon as possible (i.e enabling them to click around and have the app react). This is critical for modern web experiences trying to provide first-class user experiences on mobile.

![][image-2]

Lighthouse currently expresses TTI as a measure of when layout has stabilized, web fonts are visible and the main thread is available enough to handle user input. There are many ways of tracking TTI manually and what’s important is optimising for metrics that result in experience improvements for your users.

For libraries like React, you should be concerned by the [cost of booting up the library](8) on mobile as this can catch folks out. In [ReactHN](9), we accomplished interactivity in under **1700ms** by keeping the size and execution cost of the overall app relatively small despite having multiple views: 11KB gzipped for our app bundle and 107KB for our vendor/React/libraries bundle which looks a little like this in practice:

![][image-3]

Later, for apps with granular functionality, we’ll look at performance patterns like [PRPL](10) which nail fast time-to-interactivity through granular “route-based chunking” while taking advantage of [HTTP/2 Server Push](11) (try the [Shop](12) demo to see what we mean).

Housing.com recently shipped a React experience using a PRPL-like pattern to much praise:

![][image-4]

Housing.com took advantage of Webpack route-chunking to defer some of the bootup cost of entry pages (loading only what is needed for a route to render). For more detail see [Sam Saccone’s excellent Housing.com perf audit](13).

As did Flipkart:

Note: There are many differing views on what “time to interactive” might mean and it’s possible Lighthouse’s definition of TTI will evolve. Other ways to track it could be the first point after a navigation where you can see a 5 second window with no long tasks or the first point after a text/content paint with a 5s window with no long tasks. Basically, how soon after the page settles is it likely a user will be able to interact with the app?

Note:  While not a firm requirement, you may also improve visual completeness (Speed Index) by [optimising the critical-rendering path](14). [Tooling for critical-path CSS optimisation exists](15) and this optimisation can still have wins in a world with HTTP/2.

_If you’re new to module bundling tools like Webpack,_ [_JS module bundlers_](16) _(video) might be a useful watch._

Some of today’s JavaScript tooling makes it easy to bundle all of your scripts into a single bundle.js file that gets included in all pages.This means that a lot of the time, you’re likely loading a lot of code that isn’t required at all for the current route. Why load up 500KB of JS for a route when only 50KB will do? We should be throwing out script that isn’t conducive to shipping a fast experience for booting up a route with interactivity

![][image-5]

_Avoid serving large, monolithic bundles (like above) when just serving the minimum functionally viable code a user actually needs for a route will do._

Code-splitting is one answer to the problem of monolithic bundles. It’s the idea that by defining split-points in your code, it can be split into different files that are lazy loaded on demand. This improves startup time and help us get to being interactive sooner.

![][image-6]

Imagine using an apartment listings app. If we land on a route listing the properties in our area (route-1) — we don’t need the code for viewing the full details for a property (route-2) or scheduling a tour (route-3), so we can serve users just the JavaScript needed for the listings route and dynamically load the rest.

This idea of code-splitting by route been used by many apps over the years, but is currently referred to as “[route-based chunking](17)”. We can enable this setup for React using the Webpack module bundler.

Webpack supports code-splitting your app into chunks wherever it notices a [require.ensure()](18) being used (or in [Webpack 2](19), a [System.import](20)). These are called “split-points” and Webpack generates a separate bundle for each of them, resolving dependencies as needed.

// Defines a "split-point"  
require.ensure(\[\], function () {  
   const details = require('./Details');  
   // Everything needed by require() goes into a separate bundle  
   // require(deps, cb) is asynchronous. It will async load and evaluate  
   // modules, calling cb with the exports of your deps.  
});

When your code needs something, Webpack makes a JSONP call to fetch it from the server. This works well with React Router and we can lazy-load in the dependencies (chunks) a new route needs before rendering the view to a user.

Webpack 2 supports [automatic code-splitting with React Router](21) as it can treat System.import calls for modules as import statements, bundling imported filed and their dependencies together. Dependencies won’t collide with the initial entry in your Webpack configuration.

      import App from '../containers/App';

      function errorLoading(err) {  
        console.error('Lazy-loading failed', err);  
      }

      function loadRoute(cb) {  
        return (module) => cb(null, module.default);  
      }  
      export default {  
        component: App,  
        childRoutes: \[  
          // ...  
          {  
            path: 'booktour',  
            getComponent(location, cb) {  
              System.import('../pages/BookTour')  
                .then(loadRoute(cb))  
                .catch(errorLoading);  
            }  
          }  
        \]  
      };

Before we continue, one optional addition to your setup is [<link rel=”preload”>](22) from [Resource Hints](23). This gives us a way to declaratively fetch resources without executing them. Preload can be leveraged for preloading Webpack chunks for routes users _are likely_ to navigate to so the cache is already primed with them and they’re instantly available for instantiation.

![][image-7]

At the time of writing, preload is only implemented in [Chrome](24), but can be treated as a progressive enhancement for browsers that do support it.

Note: html-webpack-plugin’s [templating and custom-events](25) can make setting this up a trivial process with minimal changes. You should however ensure that resources being preloaded are genuinely going to be useful for your averages users journey.

Back to code-splitting — in an app using React and [React Router](26), we can use require.ensure() to asynchronously load a component as soon as ensure gets called. Btw, this needs to be shimmed in Node using the [node-ensure](27) package for anyone exploring server-rendering. Pete Hunt covers async loading in [Webpack How-to](28).

In the below example, require.ensure() enables us to lazy load routes as needed, waiting on a component to be fetched before it is used:

const rootRoute = {  
  component: Layout,  
  path: '/',  
  indexRoute: {  
    getComponent (location, cb) {  
      require.ensure(\[\], () => {  
        cb(null, require('./Landing'))  
      })  
    }  
  },  
  childRoutes: \[  
    {  
      path: 'book',  
      getComponent (location, cb) {  
        require.ensure(\[\], () => {  
          cb(null, require('./BookTour'))  
        })  
      }  
    },  
    {  
      path: 'details/:id',  
      getComponent (location, cb) {  
        require.ensure(\[\], () => {  
          cb(null, require('./Details'))  
        })  
      }  
    }  
  \]  
}

_Note: I often use the above setup with the CommonChunksPlugin (with minChunks: Infinity) so I have one chunk with common modules between my different entry points. This also_ [_minimized_](29) _running into missing Webpack runtime._

Brian Holt covers async route loading well in a [Complete Intro to React](30). Code-splitting with async routing is possible with both the current version of React Router and the [new React Router V4](31).

Here’s a tip for getting code-splitting setup even faster. In React Router, a [declarative route](32) for mapping a route “/” to a component 'App' looks like <Route path=”/” component={App}>.

React Router also supports a handy '[getComponent](33); attribute, which is similar to 'component' but is asynchronous and is **super nice** for getting code-splitting setup quickly:

<Route   
   path="stories/:storyId"   
   getComponent={(nextState, cb) => {  
   // async work to find components  
  cb(null, Stories)  
}} />

'getComponent' takes a function defining the next state (which I set to null) and a callback.

Let’s add some route-based code-splitting to [ReactHN](34). We’ll start with a snippet from our [routes](35) file — this defines require calls for components and React Router routes for each route (e.g news, item, poll, job, comment permalinks etc):

var IndexRoute = require('react-router/lib/IndexRoute')  
var App = require('./App')  
var Item = require('./Item')  
var PermalinkedComment = require('./PermalinkedComment') <--  
var UserProfile = require('./UserProfile')  
var NotFound = require('./NotFound')  
var Top = stories('news', 'topstories', 500)  
// ....

module.exports = <Route path="/" component={App}>  
  <IndexRoute component={Top}/>  
  <Route path="news" component={Top}/>  
  <Route path="item/:id" component={Item}/>  
  <Route path="job/:id" component={Item}/>  
  <Route path="poll/:id" component={Item}/>  
  <Route path="comment/:id" component={PermalinkedComment}/> <---  
  <Route path="newcomments" component={Comments}/>  
  <Route path="user/:id" component={UserProfile}/>  
  <Route path="*" component={NotFound}/>  
</Route>

ReactHN currently serve users a monolithic bundle of JS with code for _all_ routes. Let’s switch it up to route-chunking and only serve exactly the code needed for a route, starting with comment permalinks (comment/:id):

So we first delete the implicit require for the permalink component:

var PermalinkedComment = require(‘./PermalinkedComment’)

Then we take our route..

<Route path=”comment/:id” component={PermalinkedComment}/>

And update it with some declarative getComponent goodness. We’ve got our require.ensure() call to lazy-load in our route and this is all we need to do for code-splitting:

<Route  
    path="comment/:id"  
    getComponent={(location, callback) => {  
      require.ensure(\[\], require => {  
        callback(null, require('./PermalinkedComment'))  
      }, 'PermalinkedComment')  
    }}  
  />

OMG beautiful. And..that’s it. Seriously. We can apply this to the rest of our routes and run webpack. It will correctly find the require.ensure() calls and split our code as we intended.

![][image-8]

After applying declarative code-splitting to many more of our routes we can see our route-chunking in action, only loading up the code needed for a route (which we can precache in Service Worker) as needed:

![][image-9]

Reminder: A number of drop-in Webpack plugins for Service Worker caching are available:

*   [sw-precache-webpack-plugin](36) which uses sw-precache under the hood
*   [offline-plugin](37) which is used by react-boilerplate

![][image-10]

To identify common modules used across different routes and put them in a commons chunk, use the [CommonsChunkPlugin](38). It requires two script tags to be used per page, one for the commons chunk and one for the entry chunk for a route.

const CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");  
module.exports = {  
    entry: {  
        p1: "./route-1",  
        p2: "./route-2",  
        p3: "./route-3"  
    },  
    output: {  
        filename: "\[name\].entry.chunk.js"  
    },  
    plugins: \[  
        new CommonsChunkPlugin("commons.chunk.js")  
    \]  
}

The Webpack [— display-chunks flag](39) is useful for seeing what modules occur in which chunks. This helps narrow down what dependencies are being duplicated in chunks and can hint at whether or not it’s worth enabling the CommonChunksPlugin in your project. Here’s a project with multiple components that detected a duplicate Mustache.js dependency between different chunks:

![][image-11]

Webpack 1 also supports deduplication of libraries in your dependency trees using the [DedupePlugin](40). In Webpack 2, tree-shaking should mostly eliminate the need for this.

**More Webpack tips**

*   The number of require.ensure() calls in your codebase generally correlates to the number of bundles that will be generated. It’s useful to be aware of this when heavily using ensure across your codebase.
*   [Tree-shaking in Webpack2](41) will help remove unused exports. This can help keep your bundle sizes smaller.
*   Also, be careful to avoid require.ensure() calls in common/shared bundles. You might find this creates entry point references which have assumptions about the dependencies that have already been loaded.
*   In Webpack 2, System.import does not currently work with server-rendering but I’ve shared some notes about how to work around this on [StackOverflow](42).
*   If optimising for build speed, look at the [Dll plugin](43), [parallel-webpack](44) and targeted builds
*   If you need to **async** or **defer** scripts with Webpack, see [script-ext-html-webpack-plugin](45)

**Detecting bloat in Webpack builds**

The Webpack community have many web-established analysers for builds including [http://webpack.github.io/analyse/](46), [](47) [https://chrisbateman.github.io/webpack-visualizer/](48), [](49) a[n](50)d [](51) [https://alexkuz.github.io/stellar-webpack/.](52) These are handy for understanding what your largest modules are.

[**source-map-explorer**](53) (via Paul Irish) is also _fantastic_ for understanding code bloat through source maps. Look at this tree-map visualisation with per-file LOC and % breakdowns for the ReactHN Webpack bundle:

![][image-12]

You might also be interested in [**coverage-ext**](54) by Sam Saccone for generating code coverage for any webapp. This is useful for understanding how much code of the code you’re shipping down is actually being executed.

Polymer discovered an interesting web performance pattern for granularly serving apps called [PRPL](55) (see [Kevin’s I/O talk](56)). This pattern tries to optimise for interactivity and stands for:

*   (P)ush critical resources for the initial route
*   (R)ender initial route and get it interactive as soon as possible
*   (P)re-cache the remaining routes using Service Worker
*   (L)azy-load and lazily instantiate parts of the app as the user moves through the application

![][image-13]

We have to give great kudos here to the [Polymer Shop demo](57) for showing us the way on real mobile devices. Using PRPL (in this case with HTML Imports, which can take advantage of the browser’s background HTML parser). No pixels go on screen that you can’t use. Additional work here is chunked and stays interactive. We’re interactive on a real mobile device at 1.75seconds. 1.3s of JavaScript but it’s all broken up. After that it all works.

You’re hopefully on board with the benefits of breaking down applications into more granular chunks by now. When a user first visits our PWA, let’s say they go to a particular route. The server (using H/2 Push) can push down the chunks needed for just that route — these are only the pieces needed to get the application booted up. Those go into the network cache.

Once they’ve been pushed down, we’ve effectively primed the cache with the chunks we know the page will need. When the application boots up, it looks at the route and knows that what we need is already in the cache, so we get that really fast first load of our application — not just a splash screen — but the interactive content the user asked for.

The next part of this is rendering the content for the view as quickly as possible. The third is, while the user is looking at the current view, using Service Worker to start pre-caching all of the other chunks and routes the user hasn’t asked for yet and getting those all installed into the Service Worker cache.

At this point the entire application (or a lot more of it) can be available offline. When a user navigates to a different part of the application, we can lazy load the next parts of it from the Service Worker cache. There’s no network loading needed because they’re already precached. Instant loading awesomeness ahoy! ❤

PRPL can be applied to any app, as Flipkart recently demonstrated on their React stack. Apps fully using PRPL can take advantage of fast-loading using HTTP/2 server push by producing two builds that we conditionally serve depending on your browser support:

\* A bundled build optimised to minimize round-trips for servers/browsers without HTTP/2 Push support. For most of us, this is what we ship today by default.

\* An unbundled build for servers/browsers that do support HTTP/2 Push enabling a faster first-paint

This builds on some of the thinking we talked about earlier with route-chunking. With PRPL, the server and our Service Worker work together to precache resources for intactive routes. When a user navigates around your app and changes routes, we lazy-load resources for routes not cached yet and create the required views.

**tl;dr: Webpack’s require.ensure() with an async ‘getComponent’ and React Router are the lowest friction paths to a PRPL-style performance pattern**

![][image-14]

A big part of PRPL is turning the JS bundling mindset upside down and delivering resources as close to the granularity in which they are authored as possible (at least in terms of functionally independent modules). With Webpack, this is all about route-chunking which we’ve already covered.

Push critical resources for the initial route. Ideally, using [HTTP/2 Server Push](58) however don’t let this be a blocker for trying to go down a PRPL-like path. You can achieve substantially similar results to “full” PRPL in many cases without using H/2 Push, but just sending [preload headers](59) and H/2 alone.

See this production waterfall by Flipkart of their before/after wins:

![][image-15]

Webpack has support for H/2 in the form of [AggressiveSplittingPlugin](60).

AggressiveSplittingPlugin splits every chunk until it reaches the specified maxSize as we can see with a short example below:

      module.exports = {  
          entry: "./example",  
          output: {  
              path: path.join(__dirname, "js"),  
              filename: "\[chunkhash\].js",  
              chunkFilename: "\[chunkhash\].js"  
          },  
          plugins: \[  
              new webpack.optimize.AggressiveSplittingPlugin({  
                  minSize: 30000,  
                  maxSize: 50000  
              }),  
      // ...

See the official [plugin page](61) with examples for more details. [Lessons learned experimenting with HTTP/2 Push](62) and [Real World HTTP/2](63) are also worth a read.

*   Rendering initial routes: this is really up to the framework/library you’re using.
*   Pre-caching remaining routes. For caching, we rely on Service Worker. [sw-precache](64) is great for generating a Service Worker for static asset precaching and for Webpack we can use [SWPrecacheWebpackPlugin](65).
*   Lazy-load and create remaining routes on demand — require.ensure() and System.import() are your friend in Webpack land.

**Why care about static asset versioning?**

Static assets refer to our page’s static resources like scripts, stylesheets and images. When users visit our page for the first time, they need to download all of the resources used by the it. Let’s say we land on a route and the JavaScript chunks needed haven’t changed since the last time the page was visited — we shouldn’t have to re-fetch these scripts because they should already exist in the browser cache. Fewer network requests is a win for web performance.

Normally, we accomplish this by setting up an [expires header](66) for each of our files. An expires header just means that we can tell the browser to avoid making another request to the server for this file for a specific amount of time (e.g 1 year). As codebases evolve and are redeployed we want to make sure users get the freshest files without having to re-download resources if they haven’t changed.

[Cache-busting](67) accomplishes this by appending a string to filenames — it could be a build version (e.g src=”chunk.js?v=1.2.0”), a timestamp or something else. I prefer to adding a hash of the file contents to the filename (e.g chunk.d9834554decb6a8j.js) as this should always change when the contents of the file changes. MD5 hashing is commonly used in the Webpack community for this purpose which generates a ‘summary’ value 16 bytes long.

[_Long-term caching of static-assets with Webpack_](68) _is an excellent read on this topic and you should check it out. I try to cover the main points of what’s involved otherwise below._

**Asset versioning using content-hashing in Webpack**

In Webpack, asset versioning using content hashing is setup [\[chunkhash\]](https://webpack.github.io/docs/long-term-caching.html) in our Webpack config as follows:

**filename: ‘\[name\].\[chunkhash\].js’,  
chunkFilename: ‘\[name\].\[chunkhash\].js’**

We also want to make sure the normal \[name\].js and content-hashed (\[name\].\[chunkhash\].js) filenames are always correctly referenced in our HTML files. This is the difference between referencing <script src=”chunk”.js”> and <script src=”chunk.d9834554decb6a8j.js”>.

Below is a commented Webpack config sample that includes a few other plugins that smooth over getting long-term caching setup.

const path = require('path');  
const webpack = require('webpack');

// Use webpack-manifest-plugin to generate asset manifests with a mapping from source files to their corresponding outputs. Webpack uses IDs instead of module names to keep generated files small in size. IDs get generated and mapped to chunk filenames before being put in the chunk manifest (which goes into our entry chunk). Unfortunately, any changes to our code update the entry chunk including the new manifest, invalidating our caching.  
const ManifestPlugin = require('webpack-manifest-plugin');

// We fix this with chunk-manifest-webpack-plugin, which puts the manifest in a completely separate JSON file of its own.  
const ChunkManifestPlugin = require('chunk-manifest-webpack-plugin');

module.exports = {  
  entry: {  
    vendor: './src/vendor.js',  
    main: './src/index.js'  
  },  
  output: {  
    path: path.join(__dirname, 'build'),  
    filename: '\[name\].\[chunkhash\].js',  
    chunkFilename: '\[name\].\[chunkhash\].js'  
  },  
  plugins: \[  
    new webpack.optimize.CommonsChunkPlugin({  
      name: "vendor",  
      minChunks: Infinity,  
    }),  
    new ManifestPlugin(),  
    new ChunkManifestPlugin({  
      filename: "chunk-manifest.json",  
      manifestVariable: "webpackManifest"  
    }),  
    // Work around non-deterministic ordering for modules. Covered more in the long-term caching of static assets with Webpack post.  
    new webpack.optimize.OccurenceOrderPlugin()   
  \]  
};

Now that we have a build of the chunk-manifest JSON, we need to inline it into our HTML so that Webpack actually has access to it when the page boots up. So include the output of the above in a <script> tag. Automatically inlining this script in HTML can be achieved using the [html-webpack-plugin](69).

_Note: Webpack are hoping to simplify the steps required for this long-term caching setup from ~4–1 by having_ [_no shared ID range_](70)_._

To learn more about HTTP [Caching best practices](71), read Jake Archibald’s excellent write-up.

In part 3 of this series, we’ll look at [**how to get your React PWA working offline and under flaky network conditions**](72).

If you’re new to React, I’ve found [React for Beginners](73) by Wes Bos excellent.

_With thanks to Gray Norton, Sean Larkin, Sunil Pai, Max Stoiber, Simon Boudrias, Kyle Mathews and Owen Campbell-Moore for their reviews._

See the Pen [SNOW GLOBE](74) by noname ([@al-ro](75)) on [CodePen](76).

 ![Elva dressed as a fairy][image-16]


[image-1]: https://cdn-images-1.medium.com/max/2000/0*KlJk2hhZl3wyn6E4.
[image-2]: https://cdn-images-1.medium.com/max/1600/0*qfZvSxxJxPHhXXgb.
[image-3]: https://cdn-images-1.medium.com/max/2000/0*N--j53GygKHn2ViI.
[image-4]: https://cdn-images-1.medium.com/max/1600/0*55ArR_Z3qt7Az_FW.
[image-5]: https://cdn-images-1.medium.com/max/1600/0*z2tqS124xW0GDmcP.
[image-6]: https://cdn-images-1.medium.com/max/2000/0*c9rmq2rp95BN39qg.
[image-7]: https://cdn-images-1.medium.com/max/1600/0*l-XqjMw7_XX0wsxX.
[image-8]: https://cdn-images-1.medium.com/max/1600/0*glKcFK9_RLNk9AyR.
[image-9]: https://cdn-images-1.medium.com/max/1600/0*tVvolw4FTKjNFAnY.
[image-10]: https://cdn-images-1.medium.com/max/1600/0*QphlrnwHQiOsB06w.
[image-11]: https://cdn-images-1.medium.com/max/1600/0*YMvoz-W2HL3v2MIs.
[image-12]: https://cdn-images-1.medium.com/max/1600/0*D5j-Jv_FVkMigRyZ.
[image-13]: https://cdn-images-1.medium.com/max/2000/0*2XxuNsDEp1-4VuoU.
[image-14]: https://cdn-images-1.medium.com/max/1600/0*-llrY94drXMjBUW6.
[image-15]: https://cdn-images-1.medium.com/max/2000/0*-hLp_Acvig_s4Uop.
[image-16]: elva-fairy-800w.jpg


[1]: https://codepen.io/Uberche/pen/yqVgqE/
[2]: https://codepen.io/Uberche
[3]: https://codepen.io
[4]: https://www.quora.com/What-does-First-Meaningful-Paint-mean-in-Web-Performance
[5]: https://sites.google.com/a/webpagetest.org/docs/using-webpagetest/metrics/speed-index
[6]: https://www.youtube.com/watch?v=IxXGMesq_8s
[7]: https://developers.google.com/web/tools/chrome-devtools/profile/evaluate-performance/rail?hl=en
[8]: https://aerotwist.com/blog/the-cost-of-frameworks/
[9]: https://github.com/insin/react-hn
[10]: https://www.polymer-project.org/1.0/toolbox/server
[11]: https://www.igvita.com/2013/06/12/innovating-with-http-2.0-server-push/
[12]: https://shop.polymer-project.org/
[13]: https://twitter.com/samccone/status/771786445015035904
[14]: https://developers.google.com/web/fundamentals/performance/critical-rendering-path/
[15]: https://github.com/addyosmani/critical-path-css-tools#node-modules
[16]: https://www.youtube.com/watch?v=OhPUaEuEaXk
[17]: https://gist.github.com/addyosmani/44678d476b8843fd981ff8011d389724
[18]: https://webpack.github.io/docs/code-splitting.html
[19]: https://gist.github.com/sokra/27b24881210b56bbaff7
[20]: http://moduscreate.com/code-splitting-for-react-router-with-es6-imports/
[21]: https://medium.com/modus-create-front-end-development/automatic-code-splitting-for-react-router-w-es6-imports-a0abdaa491e9#.3ryyedhfc
[22]: https://www.smashingmagazine.com/2016/02/preload-what-is-it-good-for/
[23]: https://twitter.com/addyosmani/status/743571393174872064
[24]: http://caniuse.com/#feat=link-rel-preload
[25]: https://github.com/ampedandwired/html-webpack-plugin#events
[26]: https://github.com/reactjs/react-router
[27]: https://www.npmjs.com/package/node-ensure
[28]: https://github.com/petehunt/webpack-howto#9-async-loading
[29]: https://github.com/webpack/webpack/issues/368#issuecomment-247212086
[30]: https://btholt.github.io/complete-intro-to-react/
[31]: https://gist.github.com/acdlite/a68433004f9d6b4cbc83b5cc3990c194
[32]: https://github.com/ReactTraining/react-router/blob/master/docs/API.md#route
[33]: https://github.com/ReactTraining/react-router/blob/master/docs/API.md#getcomponentnextstate-callback
[34]: https://github.com/insin/react-hn
[35]: https://github.com/insin/react-hn/blob/master/src/routes.js#L36
[36]: https://github.com/goldhand/sw-precache-webpack-plugin
[37]: https://github.com/NekR/offline-plugin
[38]: https://webpack.github.io/docs/list-of-plugins.html#commonschunkplugin
[39]: https://blog.madewithlove.be/post/webpack-your-bags/
[40]: https://github.com/webpack/docs/wiki/optimization#deduplication
[41]: https://medium.com/modus-create-front-end-development/webpack-2-tree-shaking-configuration-9f1de90f3233
[42]: http://stackoverflow.com/a/39088208
[43]: https://github.com/webpack/docs/wiki/list-of-plugins
[44]: https://www.npmjs.com/package/parallel-webpack
[45]: https://github.com/numical/script-ext-html-webpack-plugin
[46]: http://webpack.github.io/analyse/
[47]: http://webpack.github.io/analyse/
[48]: https://chrisbateman.github.io/webpack-visualizer/
[49]: https://chrisbateman.github.io/webpack-visualizer/
[50]: https://chrisbateman.github.io/webpack-visualizer/
[51]: https://chrisbateman.github.io/webpack-visualizer/
[52]: https://alexkuz.github.io/stellar-webpack/
[53]: https://github.com/danvk/source-map-explorer
[54]: https://github.com/samccone/coverage-ext
[55]: https://www.polymer-project.org/1.0/toolbox/server
[56]: https://www.youtube.com/watch?v=J4i0xJnQUzU
[57]: https://shop.polymer-project.org/
[58]: https://www.igvita.com/2013/06/12/innovating-with-http-2.0-server-push/
[59]: https://www.smashingmagazine.com/2016/02/preload-what-is-it-good-for/
[60]: https://github.com/webpack/webpack/tree/master/examples/http2-aggressive-splitting
[61]: https://github.com/webpack/webpack/tree/master/examples/http2-aggressive-splitting
[62]: https://docs.google.com/document/d/1K0NykTXBbbbTlv60t5MyJvXjqKGsCVNYHyLEXIxYMv0/preview?pref=2&pli=1
[63]: https://99designs.com.au/tech-blog/blog/2016/07/14/real-world-http-2-400gb-of-images-per-day/
[64]: https://github.com/GoogleChrome/sw-precache
[65]: https://www.npmjs.com/package/sw-precache-webpack-plugin
[66]: https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=en
[67]: https://css-tricks.com/strategies-for-cache-busting-css/
[68]: https://medium.com/@okonetchnikov/long-term-caching-of-static-assets-with-webpack-1ecb139adb95
[69]: https://github.com/ampedandwired/html-webpack-plugin
[70]: https://github.com/webpack/webpack/tree/master/examples/explicit-vendor-chunk
[71]: https://jakearchibald.com/2016/caching-best-practices/
[72]: https://medium.com/@addyosmani/progressive-web-apps-with-react-js-part-3-offline-support-and-network-resilience-c84db889162c#.tcspudthd
[73]: https://goo.gl/G1WGxU
[74]: https://codepen.io/al-ro/pen/XBXWjm/
[75]: https://codepen.io/al-ro
[76]: https://codepen.io
