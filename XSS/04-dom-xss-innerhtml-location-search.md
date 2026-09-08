# LAB-4 DOM XSS in `innerHTML` Sink Using `location.search`

## LAB : [PortSwigger Lab 4 Link](https://portswigger.net/academy/labs/launch/643e053e185d7855c6c7ea780bb082fa545fdd40e72aabc79f1742eb2ad07914?referrer=%2fweb-security%2fcross-site-scripting%2fdom-based%2flab-innerhtml-sink)

## What?

This lab contains a **DOM-based cross-site scripting (XSS)** vulnerability in the blog's search functionality.

The application uses an `innerHTML` assignment to modify the contents of a `div` element using data taken from the URL's `location.search`.

The goal is to perform an XSS attack that executes the JavaScript `alert()` function.

## Where?

* **Functionality:** Blog search
* **Method:** GET
* **Source:** `location.search`
* **Sink:** `innerHTML`
* **Context:** HTML
* **Authentication:** Not required

## How did I find it?

I tested the search functionality and observed that the search input was processed on the page through the DOM.

Because the application uses data from `location.search` in an `innerHTML` assignment, the input can be interpreted as HTML instead of being treated as plain text.

This indicated a potential DOM XSS vulnerability.

## How did I verify it?

I entered the following payload into the search box:

`<img src=1 onerror=alert(1)>`

Then I clicked **Search**.

The payload executed and an alert appeared, confirming the DOM XSS vulnerability.

## What happens?

The payload creates an `img` element with an intentionally invalid `src` value:

`<img src=1 onerror=alert(1)>`

Because the image cannot be loaded, the browser triggers the `onerror` event handler.

The event handler then executes:

`alert(1)`

The execution flow is:

`location.search` → `innerHTML` → injected `<img>` element → image loading error → `onerror` → `alert(1)`

## Why does it work?

The vulnerability exists because untrusted data from `location.search` is inserted into the page using the `innerHTML` sink.

`innerHTML` causes the browser to parse the supplied value as HTML. This allows an attacker to inject an HTML element containing an executable event handler.

In this case:

`<img src=1 onerror=alert(1)>`

The invalid image source causes the `onerror` event to fire, resulting in JavaScript execution.

## Important Observation

A `<script>` payload is not necessarily effective in an `innerHTML` sink.

For example:

`<script>alert(1)</script>`

does not execute when inserted through `innerHTML` because script elements inserted using `innerHTML` are not executed by the browser.

However, HTML elements with executable event handlers, such as:

`<img src=1 onerror=alert(1)>`

can still result in JavaScript execution.

This is an important distinction when testing DOM XSS: **the sink determines which payloads are effective.**

## What can an attacker do?

If this vulnerability were exploitable against another user's browser, an attacker could potentially execute JavaScript in the security context of the vulnerable website.

Depending on the application's security controls and the victim's privileges, this could potentially lead to:

* Performing actions as the victim
* Accessing sensitive page data available to JavaScript
* Manipulating page content
* Stealing sensitive information accessible to the script
* Chaining the XSS with other application vulnerabilities

## What are the conditions/limitations?

* The application must insert attacker-controlled data into an `innerHTML` sink.
* The attacker needs a way to influence the value of `location.search`.
* The injected HTML must survive any filtering or sanitization.
* The chosen payload must be compatible with the specific HTML/DOM context.

## How should it be fixed?

* Avoid using `innerHTML` with untrusted user-controlled data.
* Use safe DOM APIs such as `textContent` when inserting plain text.
* Apply context-appropriate output encoding or sanitization when HTML is genuinely required.
* Use a strong **Content Security Policy (CSP)** as defense-in-depth.
* Validate and sanitize untrusted input where appropriate.

## What did I learn?

* `location.search` can act as a **DOM XSS source**.
* `innerHTML` is a dangerous DOM XSS sink when used with untrusted data.
* DOM XSS does not require the server to reflect or store the payload.
* `<img onerror=...>` can execute JavaScript through an image-loading error.
* `<script>` tags inserted through `innerHTML` do not execute.
* Understanding the **source → sink → execution** flow is essential when testing DOM XSS.

## What was the key obstacle?

The main challenge was understanding why different XSS payloads behave differently in an `innerHTML` sink.

The important lesson was that identifying the vulnerable sink is only the first step. The payload must also be compatible with how the browser parses and executes the injected HTML.
