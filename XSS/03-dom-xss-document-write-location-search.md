## LAB-3 DOM XSS IN `document.write` SINK USING SOURCE `location.search`

## LAB : [PortSwigger Lab 3 Link](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink)

## What?

This lab contains a **DOM-based Cross-Site Scripting (XSS)** vulnerability in the search query tracking functionality.

The application uses the JavaScript `document.write` function to write data into the page. The data comes from `location.search`, which can be controlled through the website URL.

The objective is to exploit the DOM-based XSS vulnerability to call the `alert` function.

## Where?

* **Page:** Search functionality
* **Method:** GET
* **Source:** `location.search`
* **Sink:** `document.write`
* **Context:** HTML / `<img>` attribute
* **Authentication:** Not required

## How did I find it?

First, I entered a simple random alphanumeric string into the search bar to determine where the input appeared in the page.

The search input was reflected into an `<img>` element:

```html
<img src="/resources/images/tracker.gif?searchTerms=test">
```



After inspecting the page, I identified that the search term was being inserted into the `img` element.

The important part was that the application was using data from `location.search` and writing it into the page through `document.write`.

This created a potential DOM XSS sink.

## How did I verify it?

The original input was effectively placed inside an HTML attribute:

```html
<input type="text" value="user_input_here">
```

I then tested whether I could break out of the existing HTML context and create a new HTML element.

The following payload worked:

```html
"><svg onload=alert(1)>
```

I also tested:

```html
"><script>alert(1)</script>
```

Both payloads successfully triggered the JavaScript alert.

## Testing

The basic exploitation idea was to:

1. Break out of the existing `img` attribute using `"`.
2. Close the existing HTML context using `>`.
3. Inject a new HTML element.
4. Trigger JavaScript execution.

Payload:

```html
"><svg onload=alert(1)>
```

The payload effectively changes the structure from an attribute context into a new HTML element.

## What happens?

The attack flow is:

```text
Attacker-controlled URL
        ↓
location.search
        ↓
JavaScript reads search parameter
        ↓
document.write()
        ↓
Input inserted into HTML
        ↓
" and > break out of attribute context
        ↓
<svg onload=alert(1)>
        ↓
JavaScript executes
```

Initially, the search term appeared inside the image tracking element:

```html
<img src="/resources/images/tracker.gif?searchTerms=test">
```

After injecting the payload, the attacker-controlled input escaped the intended `img` attribute and introduced a new HTML element.

The browser then processed the injected element and executed the `onload` JavaScript handler.

## Why does it work?

The vulnerability exists because attacker-controlled data from `location.search` reaches the dangerous `document.write` sink without proper encoding or sanitization.

The important concepts are:

**Source:**

```javascript
location.search
```

This is where attacker-controlled data enters the client-side JavaScript.

**Sink:**

```javascript
document.write()
```

This writes the supplied data directly into the HTML document.

The payload:

```html
"><svg onload=alert(1)>
```

uses the quote and `>` characters to escape the existing HTML attribute and then introduces a new `<svg>` element.

The `onload` event handler executes:

```javascript
alert(1)
```

when the SVG element is loaded.

Therefore:

```text
location.search → document.write → HTML parsing → JavaScript execution
```

creates the DOM XSS vulnerability.

## What can an attacker do?

Depending on the application's functionality and the victim's privileges, DOM-based XSS can potentially allow an attacker to:

* Execute arbitrary JavaScript in the victim's browser context.
* Modify webpage content.
* Perform actions on behalf of the victim.
* Access information available to JavaScript within the application's origin.
* Create convincing phishing interfaces.
* Interact with application functionality using the victim's browser session.
* Chain the XSS with other client-side or server-side vulnerabilities.

The exact impact depends on the application's security controls and the data accessible from the vulnerable origin.

## What are the conditions/limitations?

* Attacker-controlled data must reach a dangerous DOM sink.
* The vulnerable JavaScript must process the attacker-controlled `location.search` value.
* The sink must interpret the input as HTML or executable content.
* The payload must successfully escape the existing HTML context.
* Browser security controls can limit the impact.
* Different HTML contexts require different XSS payloads.

## How should it be fixed?

**Primary:** Avoid writing untrusted data directly into HTML using dangerous DOM APIs such as `document.write()`.

**Defense-in-depth:**

* Use safe DOM APIs such as `textContent` when inserting text.
* Properly encode data according to its output context.
* Avoid `document.write()` for processing user-controlled data.
* Validate and sanitize untrusted input where appropriate.
* Use safe templating mechanisms.
* Implement a strong **Content Security Policy (CSP)** as an additional layer of protection.

## What did I learn?

I learned the difference between **server-side reflected/stored XSS** and **DOM-based XSS**.

In the first two labs, the server reflected or stored the malicious input. In this lab, the vulnerability occurs in the **client-side JavaScript processing of attacker-controlled data**.

The most important concept was understanding the relationship between a **source** and a **sink**:

```text
Source → location.search
Sink   → document.write
```

I also learned that identifying the HTML context is critical. Instead of simply injecting a `<script>` tag, I first had to escape the existing `img` attribute and then introduce a new HTML element.

The working payload:

```html
"><svg onload=alert(1)>
```

demonstrated how context-breaking can turn apparently harmless input into executable HTML/JavaScript.

## What was the key obstacle?

The main challenge was understanding **where the search input was actually being inserted**.

At first, I tested the search functionality with simple text. After inspecting the page, I discovered that the input was being placed inside an `<img>` element:

```html
<img src="/resources/images/tracker.gif?searchTerms=test">
```

The turning point was recognizing that I needed to **escape the `img` attribute first** before injecting a new HTML element.

Using:

```html
">
```

allowed the payload to break out of the existing context, after which:

```html
<svg onload=alert(1)>
```

executed successfully.

This confirmed the DOM-based XSS vulnerability.
