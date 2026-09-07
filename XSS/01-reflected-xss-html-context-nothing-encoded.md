## LAB-1 REFLECTED XSS INTO HTML CONTEXT WITH NOTHING ENCODED

## LAB : [PortSwigger Lab 1 Link](https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded)

## What?

This lab contains a simple **Reflected Cross-Site Scripting (XSS)** vulnerability in the search functionality.

The application reflects user-controlled search input directly into the HTML response without encoding or sanitizing it.

The objective is to perform a cross-site scripting attack that calls the `alert` function.

## Where?

* **Page:** Search functionality
* **Method:** GET
* **Parameter:** Search query
* **Authentication:** Not required
* **Context:** HTML

## How did I find it?

The search functionality accepts user-controlled input and reflects the submitted search term back into the page.

I tested whether HTML/JavaScript could be interpreted instead of being displayed as plain text.

The search field accepted HTML tags without encoding them, indicating that the input was being placed directly into the HTML response.

## How did I verify it?

Entered the following payload into the search box:

```html
<script>alert(1)</script>
```

Then clicked **Search**.

The browser executed the JavaScript and displayed an alert dialog, confirming the reflected XSS vulnerability.

## What happens?

1. The attacker submits a malicious JavaScript payload through the search functionality.
2. The application reflects the input into the generated HTML response.
3. The application does not HTML-encode the injected characters.
4. The browser interprets the `<script>` element as executable JavaScript.
5. `alert(1)` executes in the victim's browser.

The complete attack flow is:

```text
Attacker input
      ↓
Search functionality
      ↓
Server reflects input
      ↓
HTML response
      ↓
Browser parses <script>
      ↓
JavaScript executes
```

## Why does it work?

The vulnerability exists because the application places user-controlled input directly into an **HTML context** without proper output encoding.

The payload:

```html
<script>alert(1)</script>
```

contains a valid `<script>` element. Since the application does not encode the `<` and `>` characters or otherwise sanitize the input, the browser treats the payload as HTML and executes the JavaScript.

The key issue is **unsafe handling of untrusted input before it reaches the browser**.

## What can an attacker do?

Depending on the application's functionality and the victim's privileges, XSS can potentially allow an attacker to:

* Execute arbitrary JavaScript in the victim's browser context.
* Modify the content of the webpage.
* Perform actions on behalf of the victim.
* Access data available to JavaScript within the application's origin.
* Steal sensitive information accessible through the page.
* Phish users by modifying the application's interface.
* Chain XSS with other application vulnerabilities.

The exact impact depends on the application's security controls, authentication model, and available browser-accessible data.

## What are the conditions/limitations?

* User-controlled input must reach an HTML response.
* The input must be reflected without appropriate output encoding.
* The injected content must remain executable in the browser's HTML parsing context.
* Browser security controls and application defenses can limit the impact.
* The simple `<script>` payload works here because the lab performs no encoding or filtering.

## How should it be fixed?

**Primary:** Properly encode untrusted data for the context in which it is inserted.

**Defense-in-depth:**

* Apply context-aware output encoding.
* Use safe templating mechanisms that automatically encode untrusted data.
* Avoid inserting untrusted input directly into HTML.
* Validate input where appropriate, but do not rely on input filtering as the primary XSS defense.
* Implement a strong **Content Security Policy (CSP)** as an additional layer of protection.
* Use secure coding practices for all user-controlled data rendered by the application.

## What did I learn?

I learned that reflected XSS can be extremely simple when an application directly reflects user input into HTML without encoding.

The important concept is not just the payload itself, but **where the attacker-controlled input is inserted into the response**.

In this lab, the input enters an HTML context, allowing a `<script>` element to be interpreted and executed by the browser.

This also introduced the importance of **context-aware output encoding** as the primary defense against XSS.

## What was the key obstacle?

There was no major technical obstacle in this lab.

The main learning point was recognizing that the search input was reflected directly into the HTML response without encoding. Once the HTML context was identified, the simple payload:

```html
<script>alert(1)</script>
```

was sufficient to demonstrate successful JavaScript execution.
