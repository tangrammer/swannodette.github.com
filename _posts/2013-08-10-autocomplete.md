---
layout: post
title: "autocomplete"
description: ""
category: 
tags: []
---
{% include JB/setup %}

<style>
  #ac-ex0 {
    height: 260px;
    background-color: #efefef;
    padding: 10px;
  }

  #ac-container {
    width: 562px;
  }

  #ac-ex0 input {
    width: 100%;
    padding: 5px;
    font-size: 15px;
    font-family: inconsolata;
    box-sizing: border-box;
    -moz-box-sizing: border-box;
    -webkit-box-sizing: border-box;
  }

  #ac-ex0 ul {
    width: 100%;
    background-color: white;
    margin: 0;
    font-family: inconsolata;
    border-left: 1px solid #ccc;
    border-right: 1px solid #ccc;
    box-sizing: border-box;
    -moz-box-sizing: border-box;
    -webkit-box-sizing: border-box;
  }

  #ac-ex0 li {
    list-style: none;
    padding: 0 0 0 8px;
    margin: 0;
    border-bottom: 1px solid #ccc;
  }
</style>

This is the long promised autocompleter post. It's a doozy so I've
decided to present it as a piece of literate code. If you haven't read
the first two posts I recommend going over those first.

Here's the autocompleter in action. Make sure to try the following

* Typing control characters should not trigger async request
* Losing focus via clicking or via tabs should close menu
* Keyboard based selection works
* Mouse based selection works

<div id="ac-ex0">
    <div class="ac-container">
        <input id="autocomplete" type="text"/>
        <ul id="autocomplete-menu"></ul>
    </div>
</div>

Unlike many reactive autocompleter you'll find around the web what
follows is a non-trivial autocompleter closer to the type of component
you would actually want to integrate. As we go along we'll note the
advantages over the implementation provided by jQuery UI.

First we declare our namespace. We import the async functions and
macros. We also import the components from the previous blog post, no
need to write that code again. We also import some dom helpers and
some reactive conveniences.

```
(ns blog.autocomplete.core
  (:require-macros
    [cljs.core.async.macros :refer [go alt!]]
  (:require
    [cljs.core.async :refer [>! <! alts! put! sliding-buffer chan]]
    [blog.responsive.core :as resp]
    [blog.utils.dom :as dom]
    [blog.utils.reactive :as r]))
```

We setup the url we'll use to populate our menu:

```
(def base-url
  "http://en.wikipedia.org/w/api.php?action=opensearch&format=json&search=")
```

The autocompleter requires some new interface representations - we
need hideable UI components, we need to be able to set text fields,
and we need to update the contents of a list.

```
;; -----------------------------------------------------------------------------
;; Interface representation protocols

(defprotocol IHideable
  (-hide! [view])
  (-show! [view]))

(defprotocol ITextField
  (-set-text! [field txt])
  (-text [field]))

(defprotocol IUIList
  (-set-items! [list items]))
```

We can now think about the autocompleter. If you look at the jQuery
autocompleter you'll notice a lot of logic about event suppression,
this is because you have duplicate event handling between the jQuery
autocompleter and the jQuery menu.

In this version we're going to something a bit novel, the
autocompleter will not hold onto a menu instance, it will construct
the menu on the fly as needed and we can complete avoid the problem of
event suppression as we'll just wait until the menu subprocess
completes.

Read the last one more time - *we can just wait until the menu
subprocess completes*. That is when the menu subprocess finishes we
resume where we left off.

Because some of the event handling code is in the jQuery autocompleter
all the browser quirks must be handled there. In our implementation we
have a pure process coordination core devoid of all the browser
specific insanity. It's precisely for this reason why we can
fearlessly combine all the work from the previous post with the code
in this post. We can quarantine client idiosyncracies!

Let's cover the HTML autocompleter implementation:

First we want to write complete implementation of `ITextField` for HTML inputs.

```
(extend-type js/HTMLInputElement
  ITextField
  (-set-text! [field text]
    (set! (.-value list) text))
  (-text [field]
    (.-value field)))
```

Now we want HTML `ul` tags to act as hideable ui components. So we add
concrete implementations of `IHideable` and `IUIList`.

```
(extend-type js/HTMLUListElement
  IHideable
  (-hide! [list]
    (dom/add-class! list "hidden"))
  (-show! [list]
    (dom/remove-class! list "hidden"))

  IUIList
  (-set-items! [list items]
    (->> (for [item items] (str "<li>" item "</li>"))
      (apply str)
      (dom/set-html! list))))
```

Now we cover the event handling.

Finally we put it all together.

<script type="text/javascript" src="/assets/js/ac.js"></script>