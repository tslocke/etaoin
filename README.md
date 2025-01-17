[url-webdriver]: https://www.w3.org/TR/webdriver/
[url-wiki]: https://en.wikipedia.org/wiki/Etaoin_shrdlu#Literature
[url-tests]: https://github.com/igrishaev/etaoin/blob/master/test/etaoin/api_test.clj
[url-doc]: http://grishaev.me/etaoin/
[url-slack]: https://clojurians.slack.com/messages/C7KDM0EKW/

# Etaoin

[![CircleCI](https://circleci.com/gh/igrishaev/etaoin.svg?style=svg)](https://circleci.com/gh/igrishaev/etaoin)

Pure Clojure implementation of [Webdriver][url-webdriver] protocol. Use that
library to automate a browser, test your frontend behaviour, simulate human
actions or whatever you want.

It's named after [Etaoin Shrdlu][url-wiki] -- a typing machine that became alive
after a mysteries note was produced on it.

# Table of Contents

<!-- toc -->

- [Benefits](#benefits)
- [Capabilities](#capabilities)
- [Who uses it?](#who-uses-it)
- [Documentation](#documentation)
- [Installation](#installation)
- [Getting stated](#getting-stated)
- [Querying elements](#querying-elements)
  * [Simple queries, XPath, CSS](#simple-queries-xpath-css)
  * [Map syntax for querying](#map-syntax-for-querying)
  * [Vector syntax for querying](#vector-syntax-for-querying)
  * [Advanced queries](#advanced-queries)
    + [Querying the *nth* element matched](#querying-the-nth-element-matched)
  * [Interacting with queried elements](#interacting-with-queried-elements)
- [Mouse clicks](#mouse-clicks)
- [File uploading](#file-uploading)
- [Screenshots](#screenshots)
  * [Screening elements](#screening-elements)
- [Using headless drivers](#using-headless-drivers)
- [Postmortem: auto-save artifacts in case of exception](#postmortem-auto-save-artifacts-in-case-of-exception)
- [Reading browser's logs](#reading-browsers-logs)
- [Additional parameters](#additional-parameters)
- [Eager page load](#eager-page-load)
- [Keyboard chords](#keyboard-chords)
- [File download directory](#file-download-directory)
- [Setting browser profile](#setting-browser-profile)
  * [Create and find a profile in Chrome](#create-and-find-a-profile-in-chrome)
  * [Create and find a profile in Firefox](#create-and-find-a-profile-in-firefox)
  * [Running a driver with a profile](#running-a-driver-with-a-profile)
- [Scrolling](#scrolling)
- [Working with frames and iframes](#working-with-frames-and-iframes)
- [Executing Javascript](#executing-javascript)
  * [Asynchronous scripts](#asynchronous-scripts)
- [Wait functions](#wait-functions)
- [Writing Integration Tests For Your Application](#writing-integration-tests-for-your-application)
  * [Basic fixture](#basic-fixture)
  * [Multi-Driver Fixtures](#multi-driver-fixtures)
  * [Postmortem Handler To Collect Artifacts](#postmortem-handler-to-collect-artifacts)
  * [Running Tests By Tag](#running-tests-by-tag)
  * [Check whether a file has been downloaded](#check-whether-a-file-has-been-downloaded)
- [Installing Drivers](#installing-drivers)
- [Troubleshooting](#troubleshooting)
  * [Calling maximize function throws an error](#calling-maximize-function-throws-an-error)
  * [Querying wrong elements with XPath expressions](#querying-wrong-elements-with-xpath-expressions)
  * [Clicking On Non-Visible Element](#clicking-on-non-visible-element)
  * [Unpredictable errors in Chrome when window is not active](#unpredictable-errors-in-chrome-when-window-is-not-active)
- [Contributors](#contributors)
- [Other materials](#other-materials)
- [License](#license)

<!-- tocstop -->

## Benefits

- Selenium-free: no long dependencies, no tons of downloaded jars, etc.
- Lightweight, fast. Simple, easy to understand.
- Compact: just one main module with a couple of helpers.
- Declarative: the code is just a list of actions.

## Capabilities

- Currently supports Chrome, Firefox, Phantom.js and Safari (partially).
- May either connect to a remote driver or run it on your local machine.
- Run your unit tests directly from Emacs pressing `C-t t` as usual.
- Can imitate human-like behaviour (delays, typos, etc).

## Who uses it?

- [Flyerbee](https://www.flyerbee.com/)
- [Room Key](https://www.roomkey.com/)
- [Barrick Gold](http://www.barrick.com/)
- [Doctor Evidence](http://drevidence.com/)
- [Adzerk](https://adzerk.com/)

You are welcome to submit your company into that list.

## Documentation

- [API docs][url-doc]
- [Slack channel][url-slack]
- [Unit tests][url-tests]

## Installation

Add the following into `:dependencies` vector in your `project.clj` file:

```
[etaoin "0.3.3"]
```

Works with Clojure 1.7 and above.

## Getting stated

The good news you may automate your browser directly from the REPL:

```clojure
(use 'etaoin.api)
(require '[etaoin.keys :as k])

(def driver (firefox)) ;; here, a Firefox window should appear

;; let's perform a quick Wiki session
(go driver "https://en.wikipedia.org/")
(wait-visible driver [{:id :simpleSearch} {:tag :input :name :search}])

;; search for something
(fill driver {:tag :input :name :search} "Clojure programming language")
(fill driver {:tag :input :name :search} k/enter)
(wait-visible driver {:class :mw-search-results})

;; I'm sure the first link is what I was looking for
(click driver [{:class :mw-search-results} {:class :mw-search-result-heading} {:tag :a}])
(wait-visible driver {:id :firstHeading})

;; let's ensure
(get-url driver) ;; "https://en.wikipedia.org/wiki/Clojure"

(get-title driver) ;; "Clojure - Wikipedia"

(has-text? driver "Clojure") ;; true

;; navigate on history
(back driver)
(forward driver)
(refresh driver)
(get-title driver) ;; "Clojure - Wikipedia"

;; stops Firefox and HTTP server
(quit driver)
```

You see, any function requires a driver instance as the first argument. So you
may simplify it using `doto` macros:

```clojure
(def driver (firefox))
(doto driver
  (go "https://en.wikipedia.org/")
  (wait-visible [{:id :simpleSearch} {:tag :input :name :search}])
  ;; ...
  (fill {:tag :input :name :search} k/enter)
  (wait-visible {:class :mw-search-results})
  (click :some-button)
  ;; ...
  (wait-visible {:id :firstHeading})
  ;; ...
  (quit))
```

In that case, your code looks like a DSL designed just for such purposes.

If any exception occurs during a browser session, the external process might
hang forever until you kill it manually. To prevent it, use `with-<browser>`
macros as follows:

```clojure
(with-firefox {} ff ;; additional options first, then bind name
  (doto ff
    (go "https://google.com")
    ...))
```

Whatever happens during a session, the process will be stopped anyway.

## Querying elements

Most of the functions like `click`, `fill`, etc require a query term to discover
an element on a page. For example:

```clojure
(click driver {:tag :button})
(fill driver {:id "searchInput"} "Clojure")
```

[xpath-sel]:https://www.w3schools.com/xml/xpath_syntax.asp
[css-sel]:https://www.w3schools.com/cssref/css_selectors.asp

The library supports the following query types and values.

### Simple queries, XPath, CSS

- `:active` stands for the current active element. When opening Google page for
  example, it focuses the cursor on the main search input. So there is no need
  to click on in manually. Example:

  ```clojure
  (fill driver :active "Let's search for something" keys/enter)
  ```

- any other keyword that indicates an element's ID. For Google page, it is
  `:lst-ib` or `"lst-ib"` (strings are also supported). The registry
  matters. Example:

  ```clojure
  (fill driver :lst-ib "What is the Matrix?" keys/enter)
  ```
- a string with an [XPath][xpath-sel] expression. Be careful when writing them
  manually. Check the `Troubleshooting` section below. Example:

  ```clojure
  (fill driver ".//input[@id='lst-ib'][@name='q']" "XPath in action!" keys/enter)
  ```

- a map with either `:xpath` or `:css` key with a string expression of
  corresponding syntax. Example:

  ```clojure
  (fill driver {:xpath ".//input[@id='lst-ib']"} "XPath selector" keys/enter)
  (fill driver {:css "input#lst-ib[name='q']"} "CSS selector" keys/enter)
  ```

  See the [CSS selector][css-sel] manual for more info.


### Map syntax for querying

A query might be any other map that represents an XPath expression as data. The
rules are:

- A `:tag` key represents a tag's name. It becomes `*` when not passed.
- An `:index` key expands into the trailing `[x]` clause. Useful when you need
  to select a third row from a table for example.
- Any non-special key represents an attribute and its value.
- A special key has `:fn/` namespace and expands into something specific.

Examples:

- find a form by its attributes:

  ```clojure
  (query driver {:tag :form :method :GET :class :message})
  ;; expands into .//form[@method="GET"][@class="message"]
  ```

- find a button by its text (exact match):

  ```clojure
  (query driver {:tag :button :fn/text "Press Me"})
  ;; .//button[text()="Press Me"]
  ```

- find an nth element (`p`, `a`, whatever) with "download" text:

  ```clojure
  (query driver {:fn/has-text "download" :index 3})
  ;; .//*[contains(text(), "download")][3]
  ```

- find an element that has the following class:

  ```clojure
  (query driver {:tag :div :fn/has-class "overlay"})
  ;; .//div[contains(@class, "overlay")]
  ```

- find an element that has the following classes at once:

  ```clojure
  (query driver {:fn/has-classes [:active :sticky :marked]})
  ;; .//*[contains(@class, "active")][contains(@class, "sticky")][contains(@class, "marked")]
  ```

- find all the disabled input widgets:

  ```clojure
  (query driver {:tag :input :fn/disabled true})
  ;; .//input[@disabled=true()]
  ```

### Vector syntax for querying

A query might be a vector that consists from any expressions mentioned above. In
such a query, every next term searches from a previous one recursively.

A simple example:

```clojure
(click driver [{:tag :html} {:tag :body} {:tag :a}])
```

You may combine both XPath and CSS expressions as well (pay attention at a
leading dot in XPath expression:

```clojure
(click driver [{:tag :html} {:css "div.class"} ".//a[@class='download']"])
```

### Advanced queries

#### Querying the *nth* element matched

Sometimes you may need to interact with the *nth* element of a query, for
instance when wanting to click on the second link in this example:

```html
<ul>
    <li class="search-result">
        <a href="a">a</a>
    </li>
    <li class="search-result">
        <a href="b">b</a>
    </li>
    <li class="search-result">
        <a href="c">c</a>
    </li>
</ul>
```

In this case you may either use the `:index` directive that is supported for
XPath expressions like this:

```clojure
(click driver [{:tag :li :class :search-result :index 2} {:tag :a}])
```

[nth-child]: https://www.w3schools.com/CSSref/sel_nth-child.asp

or you can use the [nth-child trick][nth-child] with the CSS expression like
this:

```clojure
(click driver {:css "li.search-result:nth-child(2) a"})
```

Finally it is also possible to obtain the *nth* element directly by using
`query-all`:

```clojure
(click-el driver (nth (query-all driver {:css "li.search-result a"}) 2))
```

Note the use of `click-el` here, as `query-all` returns an element, not a
selector that can be passed to `click` directly.

### Interacting with queried elements

To interact with elements found via a query you have to pass the query result to
either `click-el` or `fill-el`:

```clojure
(click-el driver (first (query-all driver {:tag :a})))
```

So you may collect elements into a vector and arbitrarily interact with them
at any time:

```clojure
(def elements (query-all driver {:tag :input :type :text})

(fill-el driver (first elements) "This is a test")
(fill-el driver (rand-nth elements) "I like tests!")
```

## Mouse clicks

The `click` function triggers the left mouse click on an element found with the
query term:

```clojure
(click driver {:tag :button})
```

A double click is used rarely in web yet is possible with the `double-click`
function (Chrome, Phantom.js):

```clojure
(double-click driver {:tag :dbl-click-btn})
```

There is also a bunch of "blind" clicking functions. They trigger mouse clicks
on the current mouse position:

```clojure
(left-click driver)
(middle-click driver)
(right-click driver)
```

Another bunch of functions do the same but move the mouse pointer to a specified
element before clicking on them:

```clojure
(left-click-on driver {:tag :img})
(middle-click-on driver {:tag :img})
(right-click-on driver {:tag :img})
```

A middle mouse click is useful when opening a link in a new background tab. The
right click sometimes is used to imitate a context menu in web applications.


## File uploading

Clicking on a file input button opens an OS-specific dialog that you are not
allowed to interact with using WebDriver protocol. Use the `upload-file`
function to attach a local file to a file input widget. The function takes a
selector that points to a file input and either a full path as a string or a
native `java.io.File` instance. The file should exist or you'll get an exception
otherwise. Usage example:

```clojure
(def driver (chrome))

;; open a web page that serves uploaded files
(go driver "http://nervgh.github.io/pages/angular-file-upload/examples/simple/")

;; bound selector to variable; you may also specify an id, class, etc
(def input {:tag :input :type :file})

;; upload an image with the first one file input
(def my-file "/Users/ivan/Downloads/sample.png")
(upload-file driver input my-file)

;; or pass a native Java object:
(require '[clojure.java.io :as io])
(def my-file (io/file "/Users/ivan/Downloads/sample.png"))
(upload-file driver input my-file)
```

## Screenshots

Calling a `screenshot` function dumps the current page into a PNG image on your
disk:

```clojure
(screenshot driver "page.png")             ;; relative path
(screenshot driver "/Users/ivan/page.png") ;; absolute path
```

A native Java File object is also supported:

```clojure
;; when imported as `[clojure.java.io :as io]`
(screenshot driver (io/file "test.png"))

;; native object
(screenshot driver (java.io.File. "test-native.png"))
```

### Screening elements

With Firefox, you may capture not the whole page but a single element, say a
div, an input widget or whatever. It doesn't work with other browsers for
now. Example:

```clojure
(screenshot-element driver {:tag :div :class :smart-widget} "smart_widget.png")
```

## Using headless drivers

Recently, Google Chrome and later Firefox started support a feature named
headless mode. When being headless, none of UI windows occur on the screen, only
the stdout output goes into console. This feature allows you to run integration
tests on servers that do not have graphical output device.

Ensure your browser supports headless mode by checking if it accepts `--headles`
command line argument when running it from the terminal. Phantom.js driver is
headless by its nature (it has never been developed for rendering UI).

When starting a driver, pass `:headless` boolean flag to switch into headless
mode. Note, only latest version of Chrome and Firefox are supported. For other
drivers, the flag will be ignored.

```clojure
(def driver (chrome {:headless true})) ;; runs headless Chrome
```

or

```clojure
(def driver (firefox {:headless true})) ;; runs headless Firefox
```

To check of any driver has been run in headless mode, use `headless?` predicate:

```clojure
(headless? driver) ;; true
```

Note, it will always return true for Phantom.js instances.

There are several shortcuts to run Chrome or Firefox in headless mode by
default:

```clojure
(def driver (chrome-headless))

;; or

(def driver (firefox-headless {...})) ;; with extra settings

;; or

(with-chrome-headless nil driver
  (go driver "http://example.com"))

(with-firefox-headless {...} driver ;; extra settings
  (go driver "http://example.com"))
```

There are also `when-headless` and `when-not-headless` macroses that allow to
perform a bunch of commands only if a browser is in headless mode or not
respectively:

```clojure
(with-chrome nil driver
  ...
  (when-not-headless driver
    ... some actions that might be not available in headless mode)
  ... common actions for both versions)
```

## Postmortem: auto-save artifacts in case of exception

Sometimes, it might be difficult to discover what went wrong during the last UI
tests session. A special macro `with-postmortem` saves some useful data on disk
before the exception was triggered. Those data are a screenshot, HTML code and
JS console logs. Note: not all browsers support getting JS logs.

Example:

```clojure
(def driver (chrome))
(with-postmortem driver {:dir "/Users/ivan/artifacts"}
  (click driver :non-existing-element))
```

An exception will rise, but in `/Users/ivan/artifacts` there will be three files
named by a template `<browser>-<host>-<port>-<datetime>.<ext>`:

- `firefox-127.0.0.1-4444-2017-03-26-02-45-07.png`: an actual screenshot of the
  browser's page;
- `firefox-127.0.0.1-4444-2017-03-26-02-45-07.html`: the current browser's HTML
  content;
- `firefox-127.0.0.1-4444-2017-03-26-02-45-07.json`: a JSON file with console
  logs; those are a vector of objects.

The handler takes a map of options with the following keys. All of them might be
absent.

```clojure
{;; default directory where to store artifacts; pwd is used when not passed
 :dir "/home/ivan/UI-tests"

 ;; a directory where to store screenshots; :dir is used when not passed
 :dir-img "/home/ivan/UI-tests/screenshots"

 ;; the same but for HTML sources
 :dir-src "/home/ivan/UI-tests/HTML"

 ;; the same but for console logs
 :dir-log "/home/ivan/UI-tests/console"

 ;; a string template to format a date; See SimpleDateFormat Java class
 :date-format "yyyy-MM-dd-HH-mm-ss"}
```

## Reading browser's logs

Function `(get-logs driver)` returns the browser's logs as a vector of
maps. Each map has the following structure:

```clojure
{:level :warning,
 :message "1,2,3,4  anonymous (:1)",
 :timestamp 1511449388366,
 :source nil,
 :datetime #inst "2017-11-23T15:03:08.366-00:00"}
```

Currently, logs are available in Chrome and Phantom.js only. Please note, the
message text and the source type highly depend on the browser. Chrome wipes the
logs once they have been read. Phantom.js keeps them but only until you change
the page.

## Additional parameters

When running a driver instance, a map of additional parameters might be passed
to tweak the browser's behaviour:

```clojure
(def driver (chrome {:path "/path/to/driver/binary"}))
```

Below, here is a map of parameters the library support. All of them might be
skipped or have nil values. Some of them, if not passed, are taken from the
`defaults` map.

```clojure
{;; Host and port for webdriver's process. Both are taken from defaults
 ;; when are not passed. If you pass a port that has been already taken,
 ;; the library will try to take a random one instead.
 :host "127.0.0.1"
 :port 9999

 ;; Path to webdriver's binary file. Taken from defaults when not passed.
 :path-driver "/Users/ivan/Downloads/geckodriver"

 ;; Path to the driver's binary file. When not passed, the driver discovers it
 ;; by its own.
 :path-browser "/Users/ivan/Downloads/firefox/firefox"

 ;; Extra command line arguments sent to the browser's process. See your browser's
 ;; supported flags.
 :args ["--incognito" "--app" "http://example.com"]

 ;; Extra command line arguments sent to the webdriver's process.
 :args-driver ["-b" "/path/to/firefox/binary"]

 ;; Sets browser's minimal logging level. Only messages with level above
 ;; that one will be collected. Useful for fetching Javascript logs. Possible
 ;; values are: nil (aliases :off, :none), :debug, :info, :warn (alias :warning),
 ;; :err (aliases :error, :severe, :crit, :critical), :all. When not passed,
 ;; :all is set.
 :log-level :err ;; to show only errors but not debug

 ;; Path to a custorm browser profile. See the section below.
 :profile "/Users/ivan/Library/Application Support/Firefox/Profiles/iy4iitbg.Test"

 ;; Env variables sent to the driver's process. Not processed yet.
 :env {:MOZ_CRASHREPORTER_URL "http://test.com"}

 ;; Initial window size.
 :size [1024 680]

 ;; Default URL to open. Works only in FF for now.
 :url "http://example.com"

 ;; Where to download files.
 :download-dir "/Users/ivan/Desktop"

 ;; Driver-specific options. Make sure you have read the docs before setting them.
 :capabilities {:chromeOptions {:args ["--headless"]}}}
```

## Eager page load

When you navigate to a certain page, the driver waits until the whole page has
been completely loaded. What's fine in most of the cases yet doesn't reflect the
way human beings interact with the Internet.

Change this default behavior with the `:load-strategy` option. There are three
possible values for that: `:none`, `:eager` and `:normal` which is the default
when not passed.

When you pass `:none`, the driver responds immediately so you are welcome to
execute next instructions. For example:

```clojure
(def c (chrome))
(go c "http://some.slow.site.com")
;; you'll hang on this line until the page loads
(do-something)
```

Now when passing the load strategy option:

```clojure
(def c (chrome {:load-strategy :none}))
(go c "http://some.slow.site.com")
;; no pause, acts immediately
(do-something)
```

For the `:eager` option, it works only with Firefox at the moment of adding the
feature to the library.


## Keyboard chords

There is an option to input a series of keys simultaneously. That is useful to
imitate holding a system key like Control, Shift or whatever when typing.

The namespace `etaoin.keys` carries a bunch of key constants as well as a set of
functions related to input.

```clojure
(require '[etaoin.keys :as keys])
```

A quick example of entering ordinary characters holding Shift:

```clojure
(def c (chrome))
(go c "http://google.com")

(fill-active c (keys/with-shift "caps is great"))
```

The main input gets populated with "CAPS IS GREAT". Now you'd like to delete the
last word. In Chrome, this is done by pressing backspace holding Alt. Let's do
that:

```clojure
(fill-active c (keys/with-alt keys/backspace))
```

Now you've got only "CAPS IS " in the input.

Consider a more complex example which repeats real users' behaviour. You'd like
to delete everything from the input. First, you move the caret at the very
beginning. Then move it to the end holding shit so everything gets
selected. Finally, you press delete to clear the selected text.

The combo is:

```clojure
(fill-active c keys/home (keys/with-shift keys/end) keys/delete)
```

There are also `with-ctrl` and `with-command` functions that act the same.

Pay attention, these functions do not apply to the global browser's
shortcuts. For example, neither "Command + R" nor "Command + T" reload the page
or open a new tab.

All the `keys/with-*` functions are just wrappers upon the `keys/chord` function
that might be used for complex cases.


## File download directory

To specify your own directory where to download files, pass `:download-dir`
parameter into an option map when running a driver:

```clojure
(def driver (chrome {:download-dir "/Users/ivan/Desktop"}))
```

Now, once you click on a link, a file should be put into that folder. Currently,
only Chrome and Firefox are supported.

Firefox requires to specify MIME-types of those files that should be downloaded
without showing a system dialog. By default, when the `:download-dir` parameter
is passed, the library adds the most common MIME-types: archives, media files,
office documents, etc. If you need to add your own one, override that preference
manually:

```clojure
(def driver (firefox {:download-dir "/Users/ivan/Desktop"
                      :prefs {:browser.helperApps.neverAsk.saveToDisk
                              "some-mime/type-1;other-mime/type-2"}}))
```

To check whether a file was downloaded during UI tests, see the testing section
below.

## Setting browser profile

When running Chrome or Firefox, you may specify a special profile made for test
purposes. A profile is a folder that keeps browser settings, history, bookmarks
and other user-specific data.

Imagine you'd like to run your integration tests against a user that turned off
Javascript execution or image rendering. To prepare a special profile for that
task would be a good choice.

### Create and find a profile in Chrome

1. In the right top corner of the main window, click on a user button.
2. In the dropdown, select "Manage People".
3. Click "Add person", submit a name and press "Save".
4. The new browser window should appear. Now, setup the new profile as you want.
5. Open `chrome://version/` page. Copy the file path that is beneath the
   `Profile Path` caption.

### Create and find a profile in Firefox

[ff-profile]:https://support.mozilla.org/en-US/kb/profile-manager-create-and-remove-firefox-profiles

1. Run Firefox with `-P`, `-p` or `-ProfileManager` key as the [official
   page][ff-profile] describes.
2. Create a new profile and run the browser.
3. Setup the profile as you need.
4. Open `about:support` page. Near the `Profile Folder` caption, press the `Show
   in Finder` button. A new folder window should appear. Copy its path from
   there.

### Running a driver with a profile

Once you've got a profile path, launch a driver with a special `:profile` key as
follows:

```clojure
;; Chrome
(def chrome-profile
  "/Users/ivan/Library/Application Support/Google/Chrome/Profile 2/Default")

(def chrm (chrome {:profile chrome-profile}))

;; Firefox
(def ff-profile
  "/Users/ivan/Library/Application Support/Firefox/Profiles/iy4iitbg.Test")

(def ff (firefox {:profile ff-profile}))
```

## Scrolling

The library ships a set of functions to scroll the page.

The most important one, `scroll-query` jumps the the first element found with
the query term:

```clojure
(def driver (chrome))

;; the form button placed somewhere below
(scroll-query driver :button-submit)

;; the main article
(scroll-query driver {:tag :h1})
```

To jump to the absolute position, just use `scroll` as follows:

```clojure
(scroll driver 100 600)

;; or pass a map with x and y keys
(scroll driver {:x 100 :y 600})
```

To scroll relatively, use `scroll-by` with offset values:

```clojure
;; keeps the same horizontal position, goes up for 100 pixels
(scroll-by driver 0 -100) ;; map parameter is also supported
```

There are two shortcuts to jump top or bottom of the page:

```clojure
(scroll-bottom driver) ;; you'll see the footer...
(scroll-top driver)    ;; ...and the header again
```

The following functions scroll the page in all directions:

```clojure
(scroll-down driver 200)  ;; scrolls down by 200 pixels
(scroll-down driver)      ;; scrolls down by the default (100) number of pixels

(scroll-up driver 200)    ;; the same, but scrolls up...
(scroll-up driver)

(scroll-left driver 200)  ;; ...left
(scroll-left driver)

(scroll-right driver 200) ;; ... and right
(scroll-right driver)
```

One note, in all cases the scroll actions are served with Javascript. Ensure
your browser has it enabled.

## Working with frames and iframes

While working with the page, you cannot interact with those items that are put
into a frame or an iframe. The functions below switch the current context on
specific frame:

```clojure
(switch-frame driver :frameId) ;; now you are inside an iframe with id="frameId"
(click driver :someButton)     ;; click on a button inside that iframe
(switch-frame-top driver)      ;; switches on the top of the page again
```

Frames could be nested one into another. The functions take that into
account. Say you have an HTML layout like this:

```html
<iframe src="...">
  <iframe src="...">
    <button id="the-goal">
  </frame>
</frame>
```

So you can reach the button with the following code:

```clojure
(switch-frame-first driver)  ;; switches to the first top-level iframe
(switch-frame-first driver)  ;; the same for an iframe inside the previous one
(click driver :the-goal)
(switch-frame-parent driver) ;; you are in the first iframe now
(switch-frame-parent driver) ;; you are at the top
```

To reduce number of code lines, there is a special `with-frame` macro. It
temporary switches frames while executing the body returning its last expression
and switching to the previous frame afterwards.

```clojure
(with-frame driver {:id :first-frame}
  (with-frame driver {:id :nested-frame}
    (click driver {:id :nested-button})
    42))
```

The code above returns `42` staying at the same frame that has been before
before evaluating the macros.

## Executing Javascript

To evaluate a Javascript code in a browser, run:

```clojure
(js-execute driver "alert(1)")
```

You may pass any additional parameters into the call and cath them inside a
script with the `arguments` array-like object:

```clojure
(js-execute driver "alert(arguments[2].foo)" 1 false {:foo "hello!"})
```

As the result, `hello!` string will appear inside the dialog.

To return any data into Clojure, just add `return` into your script:

```clojure
(js-execute driver "return {foo: arguments[2].foo, bar: [1, 2, 3]}"
                   1 false {:foo "hello!"})
;; {:bar [1 2 3], :foo "hello!"}
```

### Asynchronous scripts

If your script performs AJAX requests or operates on `setTimeout` or any other
async stuff, you cannot just `return` the result. Instead, a special callback
should be called against the data you'd like to achieve. The webdriver passes
this callback as the last argument for your script and might be reached with the
`arguments` array-like object.

Example:

```clojure
(js-async
  driver
  "var args = arguments; // preserve the global args
  var callback = args[args.length-1];
  setTimeout(function() {
    callback(args[0].foo.bar.baz);
  },
  1000);"
  {:foo {:bar {:baz 42}}})
```

returns `42` to the Clojure code.

To evaluate an asynchronous script, you need either to setup a special timeout
for that:

```clojure
(set-script-timeout driver 5) ;; in seconds
```

or wrap the code into a macros that does it temporary:

```clojure
(with-script-timeout driver 30
  (js-async driver "some long script"))
```

## Wait functions

The main difference between a program and a human being is that the first one
operates very fast. It means so fast, that sometimes a browser cannot render new
HTML in time. So after each action you'd better to put `wait-<something>`
function that just polls a browser until the predicate evaluates into true. Or
just `(wait <seconds>)` if you don't care about optimization.

The `with-wait` macro might be helpful when you need to prepend each action with
`(wait n)`. For example, the following form

```clojure
(with-chrome {} driver
  (with-wait 3
    (go driver "http://site.com")
    (click driver {:id "search_button"})))
```

turns into something like this:

```clojure
(with-chrome {} driver
  (wait 3)
  (go driver "http://site.com")
  (wait 3)
  (click driver {:id "search_button"}))
```

and thus returns the result of the last form of the original body.

There is another macro `(doto-wait n driver & body)` that acts like the standard
`doto` but prepend each form with `(wait n)`. For example:

```clojure
(with-chrome {} driver
  (doto-wait 1 driver
    (go "http://site.com")
    (click :this-link)
    (click :that-button)
    ...etc))
```

The final form would be something like this:

```clojure
(with-chrome {} driver
  (doto driver
    (wait 1)
    (go "http://site.com")
    (wait 1)
    (click :this-link)
    (wait 1)
    (click :that-button)
    ...etc))
```

## Writing Integration Tests For Your Application

### Basic fixture

To make your test not depend on each other, you need to wrap them into a fixture
that will create a new instance of a driver and shut it down properly at the end
if each test.

Good solution might be to have a global variable (unbound by default) that will
point to the target driver during the tests.

```clojure
(ns project.test.integration
  "A module for integration tests"
  (:require [clojure.test :refer :all]
            [etaoin.api :refer :all]))

(def ^:dynamic
  "Current driver"
  *driver*)

(defn fixture-driver
  "Executes a test running a driver. Bounds a driver
   with the global *driver* variable."
  [f]
  (with-chrome {} driver
    (binding [*driver* driver]
      (f))))

(use-fixtures
  :each ;; start and stop driver for each test
  fixture-driver)

;; now declare your tests

(deftest ^:integration
  test-some-case
  (doto *driver*
    (go url-project)
    (click :some-button)
    (refresh)
    ...
    ))
```

### Multi-Driver Fixtures

In the example above, we examined a case when you run tests against a single
type of driver. However, you may want to test your site on multiple drivers,
say, Chrome and Firefox. In that case, your fixture may become a bit more
complex:

```clojure

(def driver-type [:firefox :chrome])

(defn fixture-drivers [f]
  (doseq [type driver-types]
    (with-driver type {} driver
      (binding [*driver* driver]
        (testing (format "Testing in %s browser" (name type))
          (f))))))
```

Now, each test will be run twice in both Firefox and Chrome browsers. Please
note the test call is prepended with `testing` macro that puts driver name into
the report. Once you've got an error, you'll easy find what driver failed the
tests exactly.

### Postmortem Handler To Collect Artifacts

To save some artifacts in case of exception, wrap the body of your test into
`with-postmortem` handler as follows:

```clojure
(deftest test-user-login
  (with-postmortem *driver* {:dir "/path/to/folder"}
    (doto *driver*
      (go "http://127.0.0.1:8080")
      (click-visible :login)
      ;; any other actions...
      )))
```

Now that, if any exception occurs in that test, artifacts will be saved.

To not copy and paste the options map, declare it on the top of the module. If
you use Circle CI, it would be great to save the data into a special artifacts
directory that might be downloaded as a zip file once the build has been
finished:

```clojure
(def pm-dir
  (or (System/getenv "CIRCLE_ARTIFACTS") ;; you are on CI
      "/some/local/path"))               ;; local machine

(def pm-opt
  {:dir pm-dir})
```

Now pass that map everywhere into PM handler:

```clojure
  ;; test declaration
  (with-postmortem *driver* pm-opt
    ;; test body goes here
    )
```

Once an error occurs, you will find a PNG image that represents your browser
page at the moment of exception and HTML dump.

### Running Tests By Tag

Since UI tests may take lots of time to pass, it's definitely a good practice to
pass both server and UI tests independently from each other.

First, add `^:integration` tag to all the tests that are run inder the browser
like follows:

```clojure
(deftest ^:integration
  test-password-reset-pipeline
  (doto *driver*
    (go url-password-reset)
    (click :reset-btn)
    ...
```

Then, open your `project.clj` file and add test selectors:

```clojure
:test-selectors {:default (complement :integration)
                 :integration :integration}
```

Now, once you launch `lein test` you will run all the tests except browser
ones. To run integration tests, launch `lein test :integration`.

The main difference between a program and a human is that the first one
operates very fast. It means so fast, that sometimes a browser cannot render new
HTML in time. So after each action you need to put `wait-<something>` function
that just polls a browser checking for a predicate. O just `(wait <seconds>)` if
you don't care about optimization.

### Check whether a file has been downloaded

Sometimes, a file starts to download automatically once you clicked on a link or
just visited some page. In tests, you need to ensure a file really has been
downloaded successfully. A common scenario would be:

- provide a custom empty download folder when running a browser (see above).
- Click on a link or perform any action needed to start file downloading.
- Wait for some time; for small files, 5-10 seconds would be enough.
- Using files API, scan that directory and try to find a new file. Check if it
  matches a proper extension, name, creation date, etc.

Example:

```clojure
;; Local helper that checks whether it is really an Excel file.
(defn xlsx? [file]
  (-> file
      .getAbsolutePath
      (str/ends-with? ".xlsx")))

;; Top-level declarations
(def DL-DIR "/Users/ivan/Desktop")
(def driver (chrome {:download-dir DL-DIR}))

;; Later, in tests...
(click-visible driver :download-that-application)
(wait driver 7) ;; wait for a file has been downloaded

;; Now, scan the directory and try to find a file:
(let [files (file-seq (io/file DL-DIR))
      found (some xlsx? files)]
  (is found (format "No *.xlsx file found in %s directory." DL-DIR)))
```

## Installing Drivers

[url-webdriver]: https://www.w3.org/TR/webdriver/
[url-tests]: https://github.com/igrishaev/etaoin/blob/master/test/etaoin/api_test.clj
[url-chromedriver]: https://sites.google.com/a/chromium.org/chromedriver/
[url-chromedriver-dl]: https://sites.google.com/a/chromium.org/chromedriver/downloads
[url-geckodriver-dl]: https://github.com/mozilla/geckodriver/releases
[url-phantom-dl]: http://phantomjs.org/download.html
[url-webkit]: https://webkit.org/blog/6900/webdriver-support-in-safari-10/

This page provides instructions on how to install drivers you need to automate
your browser.

Install Chrome and Firefox browsers downloading them from the official
sites. There won't be a problem on all the platforms.

Install specific drivers you need:

- Google [Chrome driver][url-chromedriver]:

  - `brew cask install chromedriver` for Mac users
  - or download compiled binaries from the [official site][url-chromedriver-dl].
  - ensure you have at least `2.28` version installed. `2.27` and below has a
    bug related to maximizing a window (see [[Troubleshooting]]).

- Geckodriver, a driver for Firefox:

  - `brew install geckodriver` for Mac users
  - or download it from the official [Mozilla site][url-geckodriver-dl].

- Phantom.js browser:

  - `brew install phantomjs` For Mac users
  - or download it from the [official site][url-phantom-dl].

- Safari Driver (for Mac only):

  - update your Mac OS to El Captain using App Store;
  - set up Safari options as the [Webkit page][url-webkit] says (scroll down to
    "Running the Example in Safari" section).

Now, check your installation launching any of these commands. For each command,
an endless process with a local HTTP server should start.

```bash
chromedriver
geckodriver
phantomjs --wd
safaridriver -p 0
```

You may run tests for this library launching:

```bash
lein test
```

You'll see browser windows open and close in series. The tests use a local HTML
file with a special layout to validate the most of the cases.

This page holds common troubles you might face during webdriver automation.

## Troubleshooting

### Calling maximize function throws an error

Example:

```clojure
etaoin.api> (def driver (chrome))
#'etaoin.api/driver
etaoin.api> (maximize driver)
ExceptionInfo throw+: {:response {
:sessionId "2672b934de785aabb730fd19330cf40c",
:status 13,
:value {:message "unknown error: cannot get automation extension\nfrom unknown error: page could not be found: chrome-extension://aapnijgdinlhnhlmodcfapnahmbfebeb/_generated_background_page.html\n
(Session info: chrome=57.0.2987.133)\n  (Driver info: chromedriver=2.27.440174
(e97a722caafc2d3a8b807ee115bfb307f7d2cfd9),platform=Mac OS X 10.11.6 x86_64)"}},
...
```

**Solution:** just update your `chromedriver` to the last version. Tested with
2.29, works fine. People say it woks as well since 2.28.

Remember, `brew` package manager has the outdated version 2.27. You will
probably have to download binaries from the [official site][chromedriver-dl].

See the [related issue][maximize-issue] in Selenium project.

[maximize-issue]:https://github.com/SeleniumHQ/selenium/issues/3508
[chromedriver-dl]:https://sites.google.com/a/chromium.org/chromedriver/downloads

### Querying wrong elements with XPath expressions

When passing a vector-like query, say `[{:tag :p} "//*[text()='foo')]]"}]` be
careful with hand-written XPath expressions. In vector, every its expression
searches from the previous one in a loop. There is a hidden mistake here:
without a leading dot, the `"//..."` clause means to find an element from the
root of the whole page. With a dot, it means to find from the current node,
which is one from the previous query, and so forth.

That's why, it's easy to select something completely different that what you
would like. A proper expression would be: `[{:tag :p} ".//*[text()='foo')]]"}]`.

### Clicking On Non-Visible Element

Example:

```clojure
etaoin.api> (click driver :some-id)
ExceptionInfo throw+: {:response {
:sessionId "d112ce8ddb49accdae78a769d5809eae",
:status 11,
:value {:message "element not visible\n  (Session info: chrome=57.0.2987.133)\n
(Driver info: chromedriver=2.29.461585
(0be2cd95f834e9ee7c46bcc7cf405b483f5ae83b),platform=Mac OS X 10.11.6 x86_64)"}},
...
```

**Solution:** you are trying to click an element that is not visible or its
dimentions are as little as it's impossible for a human to click on it. You
should pass another selector.

### Unpredictable errors in Chrome when window is not active

**Problem:** when you focus on other window, webdriver session that is run under
Google Chrome fails.

**Solution:** Google Chrome may suspend a tab when it has been inactive for some
time. When the page is suspended, no operation could be done on it. No clicks,
Js execution, etc. So try to keep Chrome window active during test session.

## Contributors

- [Ivan Grishaev](https://github.com/igrishaev)
- [Adam Frey](https://github.com/AdamFrey)
- [JW Koelewijn](https://github.com/jwkoelewijn)
- [Miloslav Nenadál](https://github.com/nenadalm)
- [Aleh Atsman](https://github.com/atsman)
- [Marco Molteni](https://github.com/marco-m)
- [Maxim Stasenkov](https://github.com/nebesnytihohod)

The project is open for your improvements and ideas. If any of unit tests fall
on your machine please submit an issue giving your OS version, browser and
console output.

## Other materials

[ui-test]:http://grishaev.me/en/ui-test
[stream]:https://www.youtube.com/watch?v=cLL_5rETLWY

- [Thoughts on UI tests][ui-test]. My blog-post about some pitfalls that might
  occur when testing UI.
- [Live-coding session][stream] where I work on some of the Etaoin issues.

## License

Copyright © 2017 Ivan Grishaev.

Distributed under the Eclipse Public License either version 1.0 or (at your
option) any later version.
