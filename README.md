**[API docs](http://ptaoussanis.github.io/timbre/)** | **[CHANGELOG](https://github.com/ptaoussanis/timbre/blob/master/CHANGELOG.md)** | [contact & contributing](#contact--contribution) | [other Clojure libs](https://www.taoensso.com/clojure-libraries) | [Twitter](https://twitter.com/#!/ptaoussanis) | current [semantic](http://semver.org/) version:

```clojure
[com.taoensso/timbre "2.7.1"] ; Stable
[com.taoensso/timbre "3.0.0-beta1"] ; Development, non-breaking - see CHANGELOG for details
```

# Timbre, a (sane) Clojure logging & profiling library

Logging with Java can be maddeningly, unnecessarily hard. Particularly if all you want is something *simple that works out-the-box*. Timbre is an attempt to bring functional, Clojure-y goodness to all your logging needs. **No XML!**

## What's in the box™?
 * Small, uncomplicated **all-Clojure** library.
 * **Super-simple map-based config**: no arcane XML or properties files!
 * **Low overhead** with dynamic logging level.
 * **No overhead** with compile-time logging level. (v2.6+)
 * Flexible **fn-centric appender model** with **middleware**.
 * Sensible built-in appenders including simple **email appender**.
 * Tunable **rate limit** and **asynchronous** logging support.
 * Robust **namespace filtering**.
 * **[tools.logging](https://github.com/clojure/tools.logging) support** (optional).
 * Dead-simple, logging-level-aware **logging profiler**.

## Getting started

### Dependencies

Add the necessary dependency to your [Leiningen](http://leiningen.org/) `project.clj` and use the supplied ns-import helper:

```clojure
[com.taoensso/timbre "2.7.1"] ; project.clj

(ns my-app (:require [taoensso.timbre :as timbre])) ; Your ns
(timbre/refer-timbre) ; Provides useful Timbre aliases in this ns
```

The `refer-timbre` call is a convenience fn and executes:
```clojure
(require '[taoensso.timbre :as timbre
           :refer (log  trace  debug  info  warn  error  fatal  report
                   logf tracef debugf infof warnf errorf fatalf reportf
                   spy logged-future with-log-level)])
(require '[taoensso.timbre.utils :refer (sometimes)])
(require '[taoensso.timbre.profiling :as profiling :refer (pspy profile defnp)])
```

### Logging

By default, Timbre gives you basic print output to `*out*`/`*err*` at a `debug` logging level:

```clojure
(info "This will print")
=> nil
%> 2012-May-28 17:26:11:444 +0700 localhost INFO [my-app] - This will print

(spy :info (* 5 4 3 2 1))
=> 120
%> 2012-May-28 17:26:14:138 +0700 localhost INFO [my-app] - (* 5 4 3 2 1) 120

(trace "This won't print due to insufficient logging level")
=> nil
```

There's little overhead for checking logging levels:

```clojure
(time (trace (Thread/sleep 5000)))
%> "Elapsed time: 0.054 msecs"

(time (when false))
%> "Elapsed time: 0.051 msecs"
```

And _no_ overhead when using a compile-time logging level (set `TIMBRE_LOG_LEVEL` environment variable):

```clojure
(time (dotimes [_ 1000000000] (trace (Thread/sleep 5000))))
%> "Elapsed time: 387.159 msecs"

(time (dotimes [_ 1000000000] nil))
%> "Elapsed time: 389.231 msecs"
```

First-argument exceptions generate a stack trace:

```clojure
(info (Exception. "Oh noes") "arg1" "arg2")
%> 2012-May-28 17:35:16:132 +0700 localhost INFO [my-app] - arg1 arg2
java.lang.Exception: Oh noes
            NO_SOURCE_FILE:1 my-app/eval6409
          Compiler.java:6511 clojure.lang.Compiler.eval
          <...>
```

### Configuration

Configuring Timbre is a breeze. Let's check out (some of) the defaults:

```clojure
@timbre/config
=>
{:ns-whitelist [] ; Use patterns like "my-app.*"
 :ns-blacklist []

 ;; Fns to transform/filter appender fn args before dispatching to appenders:
 :middleware []

 :timestamp-pattern "yyyy-MMM-dd HH:mm:ss ZZ" ; SimpleDateFormat pattern
 :timestamp-locale  nil ; A Locale object, or nil

 :appenders
 {:standard-out        { <...> }
  :spit                { <...> }
  <...> }

 :shared-appender-config {}}
```

Easily adjust the current logging level:

```clojure
(timbre/set-level! :warn)
```

And the default timestamp formatting for log messages:

```clojure
(timbre/set-config! [:timestamp-pattern] "yyyy-MMM-dd HH:mm:ss ZZ")
(timbre/set-config! [:timestamp-locale] (java.util.Locale/GERMAN))
```

Filter logging output by namespaces:
```clojure
(timbre/set-config! [:ns-whitelist] ["some.library.core" "my-app.*"])
```

**The source code includes a fully-annotated example config** (as `timbre/example-config`) that gives further details on these and other features.

### Built-in appenders

#### File appender

```clojure
(timbre/set-config! [:appenders :spit :enabled?] true)
(timbre/set-config! [:shared-appender-config :spit-filename] "/path/my-file.log")
```

#### Email ([Postal](https://github.com/drewr/postal)) appender

```clojure
;; [com.draines/postal "1.9.2"] ; Add to project.clj dependencies
;; (:require [taoensso.timbre.appenders (postal :as postal-appender)]) ; Add to ns

(timbre/set-config! [:appenders :postal] postal-appender/postal-appender)
(timbre/set-config! [:shared-appender-config :postal]
                    ^{:host "mail.isp.net" :user "jsmith" :pass "sekrat!!1"}
                    {:from "me@draines.com" :to "foo@example.com"})

;; Rate limit to one email per message per minute
(timbre/set-config! [:appenders :postal :limit-per-msecs] 60000)

;; Make sure emails are sent asynchronously
(timbre/set-config! [:appenders :postal :async?] true)
```

#### Other included appenders

A number of appenders are included out-the-box for: Android, Carmine (Redis), IRC, sockets, MongoDB, and rotating files.

These are all located in the `taoensso.timbre.appenders.x` namespaces - please see the relevant docstrings for details.

### Custom appenders

Writing a custom appender is (really) very easy:

```clojure
(timbre/set-config!
 [:appenders :my-appender]
 {:doc        "Hello-world appender"
  :min-level  :debug
  :enabled?   true
  :async?     false
  :rate-limit nil
  :fn (fn [{:keys [ap-config level throwable output] :as args}]
        (when-not (:my-production-mode? ap-config)
          (println "Hello world!:" output)))
```

And because appender fns are just regular Clojure fns, you have *unlimited power*: write to your database, send a message over the network, check some other state (e.g. environment config) before making a choice, etc.

See the `timbre/example-config` annotated code for lots more information on appenders.

## Profiling

The usual recommendation for Clojure profiling is: use a good **JVM profiler** like [YourKit](http://www.yourkit.com/), [JProfiler](http://www.ej-technologies.com/products/jprofiler/overview.html), or [VisualVM](http://docs.oracle.com/javase/6/docs/technotes/guides/visualvm/index.html).

And these certainly do the job. But as with many Java tools, they can be a little hairy and often heavy-handed - especially when applied to Clojure. Timbre includes an alternative.

Wrap forms that you'd like to profile with the `p` macro and give them a name:

```clojure
(defn my-fn
  []
  (let [nums (vec (range 1000))]
    (+ (p :fast-sleep (Thread/sleep 1) 10)
       (p :slow-sleep (Thread/sleep 2) 32)
       (p :add  (reduce + nums))
       (p :sub  (reduce - nums))
       (p :mult (reduce * nums))
       (p :div  (reduce / nums)))))

(my-fn)
=> 42
```

The `profile` macro can now be used to log times for any wrapped forms:

```clojure
(profile :info :Arithmetic (dotimes [n 100] (my-fn)))
=> "Done!"
%> 2012-Jul-03 20:46:17 +0700 localhost INFO [my-app] - Profiling my-app/Arithmetic
              Name  Calls       Min        Max       MAD      Mean  Total% Total
 my-app/slow-sleep    100       2ms        2ms      31μs       2ms      57 231ms
 my-app/fast-sleep    100       1ms        1ms      27μs       1ms      29 118ms
        my-app/add    100      44μs        2ms      46μs     100μs       2 10ms
        my-app/sub    100      42μs      564μs      26μs      72μs       2 7ms
        my-app/div    100      54μs      191μs      17μs      71μs       2 7ms
       my-app/mult    100      31μs      165μs      11μs      44μs       1 4ms
       Unaccounted                                                       6 26ms
             Total                                                     100 405ms
```

You can also use the `defnp` macro to conveniently wrap whole fns.

It's important to note that Timbre profiling is fully **logging-level aware**: if the  level is insufficient, you *won't pay for profiling*. Likewise, normal namespace filtering applies. (Performance characteristics for both checks are inherited from Timbre itself).

And since `p` and `profile` **always return their body's result** regardless of whether profiling actually happens or not, it becomes feasible to use profiling more often as part of your normal workflow: just *leave profiling code in production as you do for logging code*.

A simple **sampling profiler** is also available: `taoensso.timbre.profiling/sampling-profile`.

## This project supports the CDS and ![ClojureWerkz](https://raw.github.com/clojurewerkz/clojurewerkz.org/master/assets/images/logos/clojurewerkz_long_h_50.png) goals

  * [CDS](http://clojure-doc.org/), the **Clojure Documentation Site**, is a **contributer-friendly** community project aimed at producing top-notch, **beginner-friendly** Clojure tutorials and documentation. Awesome resource.

  * [ClojureWerkz](http://clojurewerkz.org/) is a growing collection of open-source, **batteries-included Clojure libraries** that emphasise modern targets, great documentation, and thorough testing. They've got a ton of great stuff, check 'em out!

## Contact & contribution

Please use the [project's GitHub issues page](https://github.com/ptaoussanis/timbre/issues) for project questions/comments/suggestions/whatever **(pull requests welcome!)**. Am very open to ideas if you have any!

Otherwise reach me (Peter Taoussanis) at [taoensso.com](https://www.taoensso.com) or on Twitter ([@ptaoussanis](https://twitter.com/#!/ptaoussanis)). Cheers!

## License

Copyright &copy; 2012, 2013 Peter Taoussanis. Distributed under the [Eclipse Public License](http://www.eclipse.org/legal/epl-v10.html), the same as Clojure.