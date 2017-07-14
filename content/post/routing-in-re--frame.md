+++
title = "routing in re-frame"
draft = true
date = "2017-07-09T14:22:24-04:00"
tags = ["re-frame", "ClojureScript", "UI", "routing"]
+++


Almost all guides to front-end frameworks start with discussing data-binding and rightly so as it's biggest
differentiator between them. Typically a concept that is core to building a client-side apps is left unaddressed, and
that is routing. The re-frame Leiningen [template](https://github.com/Day8/re-frame-template) does include an option to
add basic routing to a newly scaffolded application. This guide is meant to help users who are staring a new re-frame
application and wish to add routing to it.

I've created a basic re-frame app for this guide to demonstrate the concepts I discuss here. The demo project displays a
list of the what are considered the worst songs according to
[Wikipedia](https://en.wikipedia.org/wiki/List_of_music_considered_the_worst#Songs). It can be found on
[GitHub](https://github.com/jasich/re-frame-routing-demo).

![demo](https://github.com/jasich/re-frame-routing-demo/raw/master/re-frame-routing-demo.gif)

## bidi & pushy

The official re-frame template uses [secretary](https://github.com/gf3/secretary) in combination with the [Google
Closure Library](https://github.com/google/closure-library) to provide routing and history functionality to apps. For
this guide I've chosen to use a combination of [bidi](https://github.com/juxt/bidi) for client-side routing and
[pushy](https://github.com/kibu-australia/pushy) for HTML5 pushState. I've found that these two libraries offer some key
features (like bi-directional routing & hashless URLs) that you don't get if you just use what's provided in the
template.

<pre><code>;; project.clj
(defproject re-frame-routing-demo "0.1.0-SNAPSHOT"
  :dependencies [[org.clojure/clojure "1.9.0-alpha16"]
                 [org.clojure/clojurescript "1.9.562"]
                 [reagent "0.6.0"]
                 [re-frame "0.9.2"]
                 [bidi "2.0.17"]
                 [kibu/pushy "0.3.7"]])
</code></pre>

## route definition

You'll want to create some kind of module to keep your routing code if in you haven't already had one created by the
template. Below is an example of you how you'd define routes in bidi. It contains a default route which lists all songs,
a show route for an individual song, and a route for everything that does not match.


<pre><code>;; routes.cljs
(def routes ["/" [[""       :home]           ;; "/"
                  ["songs/" {[:id] :songs}]  ;; "/songs/33"
                  [true     :not-found]]])   ;; everything else
</code></pre>

**note**: Using `true` as a route value will always match, so you'll want to define that route last. If you were to use
a hash to define your routes (which bidi supports) you'll probably run into routing issues, since they don't guarantee
order.

## matching & dispatching routes

You'll need a way to tie the routes to the current URL and also notify re-frame that it's time to update views. The
library we'll use for HTML5 pushState, pushy, requires you to provide a function that takes a URL to match to a route
and a function to handle the route matches. Once these two functions are registered you can have pushy start listening
to browser navigation.

<pre><code>;; routes.cljs
(defn- parse-url [url]
  (bidi/match-route routes url))

(defn- dispatch-route [matched-route]
  (let [handler (:handler matched-route)
        params  (:route-params matched-route)]
    (re-frame/dispatch [:set-active-panel handler params])))

(def history (pushy/pushy dispatch-route parse-url))

;; call this when re-frame is initializing to start pushy
(defn app-routes []
  (pushy/start! history))
</code></pre>

The `parse-url` function is rather straightforward, it just calls bidi to parse & match a given URL. The
`dispatch-route` will then dispatch a re-frame event once a route is matched and provide any kind of routing parameters.

## preloading data

A common task when loading routes is to also load associated data for that view from a server. To do this we can
associate the name of a route to re-frame event, which will fetch the needed data.

To start with we'll define a hash to associate the name of a route with the name of an event which will load  the necessary
data.

<pre><code>;; routes.cljs
;;
;; dispatch `:fetch-songs` when :home route is loading
(def preload {:home [:fetch-songs]})
</code></pre>

The value associated with the route name is something that we'll want to dispatch when a route match occurs. The code
below shows that if we find a valid "preload" value for a route we'll add it to the effects of our re-frame event
handler for dispatching. This code also shows that on route matches we'll update the current panel (read: view) and
update any received route parameters in the database.

<pre><code>;; events.cljs
(re-frame/reg-event-fx
  :set-active-panel
  (fn [{:keys [db]} [_ active-panel route-params]]
    (let [event-name (get routes/preload active-panel)
          route-id   (:id route-params)
          event      (if event-name [event-name route-id])]
      (if event
        {:db  (-> db
                (assoc :active-panel active-panel)
                (assoc :route-id route-id))
         :dispatch event}
        {:db  (-> db
                (assoc :active-panel active-panel)
                (assoc :route-id route-id))}))))
</code></pre>

## loading views

Whenever a route changes, our re-frame event handler will update the `:active-panel` key in the database. To update the
UI we'll subscribe to this database value and load the associated panel in to the UI.

<pre><code>;; views.cljs
(defn- panels [panel-name]
  (case panel-name
    :home       [home]
    :songs      [song]
    [not-found]))

(defn- show-panel [panel-name]
  [panels panel-name])

(defn main-panel []
  (let [active-panel (re-frame/subscribe [:active-panel])]
    (fn []
      [:div
        [:div.container
          [show-panel @active-panel]]])))
</code></pre>

## changing routes

Sometimes you may want to change routes programmatically. An example of this would be after a user logs into your site
successfully you want to redirect them to a specific page. This is really easy to do with pushy, just call `set-token!`.

<pre><code>;; routes.cljs
(defn change-route [route]
  (pushy/set-token! history route))
</code></pre>

To take advantage of this functionality just register an effect handler for changing routes that calls the
`change-route` function.

<pre><code>;; events.cljs
(re-frame/reg-fx
 :change-route
 (fn [value]
   (routes/change-route value)))
</code></pre>

Once registered you can start using `:change-route` effect handler to change routes when a re-frame event occurs.

<pre><code>;; events.cljs
(re-frame/reg-event-fx
  :login-successful
  (fn [{:keys [db]} [_ response]]
    {:db (assoc db :authenticated? true)
     :change-route "/"}))
</code></pre>

## bi-directional routing

Another cool thing that bidi gives you is bi-directional routing. This means that instead of hard coding paths in our
links we can have bidi generate them.

In `routes.cljs` we can define some helper methods now to help us generate paths. Suppose we want to redirect the user
to their account page after a login.

<pre><code>;; routes.cljs
(defn account-path-for-user [user-id]
  (bidi/path-for routes :accounts :id user-id))
</code></pre>

Our `:login-successful` event handler might look like this:

<pre><code>;; events.cljs
(re-frame/reg-event-fx
  :login-successful
  (fn [{:keys [db]} [_ response]]
    (let [user-id (fn-to-get-userid)]
      {:db (assoc db :authenticated? true)
       :change-route (routes/account-path-for-user user-id)})))
</code></pre>
