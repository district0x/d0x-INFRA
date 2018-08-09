# re-mount

re-mount is very simple yet powerful modularisation pattern for application's UI used by [district0x](https://district0x.io/). 
All that is required are 2 widely used Clojurescript libraries [re-frame](https://github.com/Day8/re-frame) and 
[mount](https://github.com/tolitius/mount). 

The pattern is not "buy all or nothing" solution for your re-frame app. You're free to use just a single module in your app and 
keep rest of code untouched, or you can go modules all the way down, or anything in between. 

## Problem
Libraries are obviously very useful for UI development. Since developers discovered that storing all application's 
state in one place is tremendously useful for UI development, libraries fragmenting application's state became largely unpopular in
UI world. The fact is though, that stateful libraries can provide higher-level abstractions and more out-of-the-box
solutions than stateless ones. Ideally, we want libraries encapsulating some part of state and providing 
consistent API to access it, while the whole state is still being stored in one place. 

## Solution
Mount and re-frame perfectly complement each other in providing consistent API for "stateful" libraries aka modules.
 
These are 4 state related lifecycle steps important for UI development. 

* Initialisation - provided by `mount/start`
* State Read - provided by re-frame `subscriptions`
* State Write - provided by re-frame `events`
* Cleanup (often not needed) - provided by `mount/stop` 

To demonstrate how this actually works in practice, let's create super simple module that provides nothing 
else but current time as state. 

Note, since re-mount module pattern is very unrestricted, modules vary in complexity widely from the simplest ones, that can
provide nothing but current time ([district.ui.now](https://github.com/district0x/district-ui-now)) to complex ones 
providing complete GraphQL solution ([district-ui-graphql](https://github.com/district0x/district-ui-graphql)).  

Full list of re-mount modules made by district0x can be found [here](https://github.com/search?q=topic%3Adistrict-ui-module+org%3Adistrict0x&type=Repositories). 

## Namespace Structure
Let's call our module `cool-dev.ui.now`. You can name re-mount module however you want, but the convention we use 
is [brandname].ui.[modulename]. 

Namespace structure of your module isn't anyhow restricted, as long as you document your module you can use basically
any structure, but this is convention we use, which is very simple and intuitive. 

* `cool-dev.ui.now` - Will contain mount module definition
* `cool-dev.ui.now.events` Will contain re-frame events
* `cool-dev.ui.now.subs` Will contain re-frame subscriptions
* `cool-dev.ui.now.queries` Will contain queries - pure functions for working with re-frame db

## cool-dev.ui.now
In this namespace we're going to define mount module: 

```clojure
(ns cool-dev.ui.now
  (:require
    [cool-dev.ui.now.events :as events]
    [mount.core :as mount :refer [defstate]]
    [re-frame.core :refer [dispatch-sync]]))

(defn start [opts]
  (dispatch-sync [::events/start opts])
  opts)
  
(defn stop []
  (dispatch-sync [::events/stop]))  

(defstate now
  :start (start (:now (mount/args)))
  :stop (stop))
```

So what we see here, is that on module start, we synchronously dispatch module's start event, so we can initialise
re-frame db with some data or run other effects, all before re-frame application starts. dispatch-sync is important, 
so you don't miss any event happening prior your start event. For start function, we also grab mount args under the 
key `:now`, so a developer can pass some configuration options to this module. 

Let's look how other developers would use this module with others in their re-frame application:

```clojure
(ns some.great.app
  (:require
    [cool-dev.ui.now]
    [other-dev.ui.other-module]
    [yet-another-dev.ui.yet-another-module]
    [some.great.app.home-page :refer [home-page]]))

(defn ^:export init []
  (-> (mount/with-args
        {:now {:some-param 1}
         :other-module {:other-param "abc"}
         :yet-another-dev {:yet-another-param "xyz"}})
    (mount/start))
  (r/render [home-page] (.getElementById js/document "app")))
```

Here we see how a developer would use three 3rd party re-mount modules, passing to each one its config parameters.
Developer doesn't need to care how each module initialises and how it uses re-frame db. Initialisation API is nicely unified. 
In fact, even last line `(r/render ...)` can be solved by re-mount module and it already is! ([district-ui-reagent-render](https://github.com/district0x/district-ui-reagent-render)),
but I didn't want to confuse you at start. Then the whole application bootstrap would be just single `mount/start`
without any copy-paste initialisation code. 

Now let's continue with implementation of our module. 

## cool-dev.ui.now.events
We're going to put here all re-frame events related to this module. 

```clojure
(ns cool-dev.ui.now.events
  (:require
    [cljs-time.core :as t]
    [district.ui.now.queries :as queries]
    [district0x.re-frame.interval-fx]
    [re-frame.core :refer [reg-event-fx trim-v]]))

(def interceptors [trim-v])

(reg-event-fx
  ::start
  interceptors
  (fn [{:keys [:db]} []]
    (merge
      {:db (queries/assoc-now db (t/now))
       :dispatch-interval {:dispatch [::update-now]
                           :id ::update-now
                           :ms 1000}})))

(reg-event-fx
  ::update-now
  interceptors
  (fn [{:keys [:db]}]
    {:db (queries/assoc-now db (t/now))}))


(reg-event-fx
  ::stop
  interceptors
  (fn [{:keys [:db]}]
    {:db (queries/dissoc-now db)
     :clear-interval {:id ::update-now}}))
```

These are all events this module needs. 

`::start` event is only run once at the bootstrap. It initialises db with current time and sets up interval for 
recurring event. Notice, we modify db only with functions from queries namespace. This makes events code cleaner, creates 
better separation of concerns, so these events or any other events don't need to be familiar with internal structure of the 
state. 

`::update-now` is simple event that is run each second

`::stop` is cleanup event that cleans this module's data from state and stops recurring interval. An application may 
or may not make us of cleanup event, but it's good practice to always provide it. 

Into this namespace we could put more events that are supposed to be used by app developer, sort of as a public API.
Re-frame events have one big advantage over regular functions: They can be hooked into. You can use [re-frame-forward-events-fx](https://github.com/Day8/re-frame-forward-events-fx) 
or [re-frame-async-flow-fx](https://github.com/Day8/re-frame-async-flow-fx) and listen to any module's event and 
associate more events with it. This is very powerful tool, which you can use to create new modules that are extending
functionality of underlying modules by reacting to their events and performing more actions. You can't globally extend regular 
function with more functionality if you need. You'd have to do pull request to the original library.
Thanks to re-frame events, re-mount modules often work as plug-n-play, where literally nothing else 
needs to be done, except including the namespace in your code and it extends code with new functionality automatically.  
Simple example of this are [district-ui-router](https://github.com/district0x/district-ui-router) and [district-ui-router-google-analytics](https://github.com/district0x/district-ui-router-google-analytics)
modules. `district-ui-router` is module, that fully handles app's routing. When `district-ui-router-google-analytics`
is included in code it automatically listens to `district-ui-router` events and reports page views to Google Analytics. 
If you prefer different service, you can simply create different module and still use `district-ui-router`.   
 
Pro tip: When designing your module always keep in mind how other developers might want to build on top of your 
module and therefore you should split complex operations into multiple events so they can hook into it at right step. 

## cool-dev.ui.now.subs
Into this namespace we put re-frame subscriptions. Provided subscriptions are another type of module's public API. 
API that's supposed to be used inside reagent components. 

```clojure
(ns cool-dev.ui.now.subs
  (:require
    [district.ui.now.queries :as queries]
    [re-frame.core :refer [reg-sub]]))

(reg-sub
  ::now
  queries/now)

```

We simply expose just one very simple subscription `::now`, that returns current time, updated each second.
Again, we're just passing this to function in queries namespace, so subscriptions don't have to deal with state 
structure. Also notice, how we're using double colon for namespacing names. This is also very good convention, so an application
with many modules don't have name conflicting problems. Namespaced keywords can be aliased same was as namespaces, so
a developer would use subscription like this: 

```clojure
(ns some.great.app.home-page
  (:require
    [cool-dev.ui.now.subs :as now-subs]
    [re-frame.core :refer [subscribe]]))
  
(defn home-page []
  [:div "Now: " @(subscribe [::now-subs/now])])
```

## cool-dev.ui.now.queries
Queries are pure functions operating upon re-frame db. State-changing functions should always return new re-frame db.
Queries are also part of module's public API. API that's supposed to be used in events or subscriptions.  

```clojure
(ns district.ui.now.queries)

(def db-key :cool-dev.ui.now)

(defn now [db]
  (get-in db [db-key :now]))

(defn assoc-now [db now]
  (assoc-in db [db-key :now] now))

(defn dissoc-now [db]
  (dissoc db db-key))
``` 

As you can see, it's all very straightforward operations, well, because our module is very simple. Another good 
convention is to define `db-key` at the top, which will serve as path base for storing module's state inside global
state. Since we want to be good citizen in modules space, we shouldn't pollute root of global state with confusing 
keys. Therefore it's good to name your `db-key` same as your module name.  

And that's it! Our module is pretty much ready. No one ever has to reinvent "now" functionality in their re-frame app again!
Some modules might include also namespaces such as `cool-dev.ui.now.utils` for utils functions, or `cool-dev.ui.now.spec`
for spec validating configuration options and input into events. 

A re-mount module can also provide reagent components closely related to module's logic. In our example we could be 
providing component with namespace: `cool-dev.ui.component.current-time`.  

More importantly, since re-mount modularisation pattern creates such a good encapsulation, where each module sits in own repository, 
we highly recommend you to create tests for each of your module. It'll make maintainability of your modules much easier. 

If you made it this far, we wholeheartedly appreciate that! If you happen to create a re-mount module, definitely
let us know! You can create issue in this repository, or contact district0x team other way. We hope to see many 
re-mount modules created by Clojurescript community, so we stop reinventing wheels in our re-frame apps :) 

Thanks! 
