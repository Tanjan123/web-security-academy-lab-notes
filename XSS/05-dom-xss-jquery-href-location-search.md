# LAB-5 DOM XSS in jQuery `href` Attribute Sink Using `location.search`

## LAB : [PortSwigger Lab 5 Link](https://portswigger.net/academy/labs/launch/fde8784b9c158cac28b9d1a29d6331d2a28cf38742e1c59454be9a1fe37cff54?referrer=%2fweb-security%2fcross-site-scripting%2fdom-based%2flab-jquery-href-attribute-sink)

## What?

This lab contains a **DOM-based cross-site scripting (XSS)** vulnerability in the **Submit feedback** page.

The application uses jQuery's `$` selector function to find an anchor element and changes its `href` attribute using data from `location.search`.

The goal is to make the **Back** link execute `alert(document.cookie)`.

## Where?

* **Page:** Submit feedback
* **Parameter:** `returnPath`
* **Source:** `location.search`
* **Sink:** Anchor `href` attribute
* **Library:** jQuery
* **Context:** URL / `href`
* **Authentication:** Not required

## How did I find it?

When clicking the **Submit feedback** button, the application redirects to the feedback page with:

`returnPath=/`

in the URL.

For example:

`https://0a9a0087046ea8f78020fd7f005200aa.web-security-academy.net/feedback?returnPath=/`

The **Back** button normally uses this value to redirect back to the home page.

I then changed the parameter to include a random value: `returnPath=/abcd`



and clicked the Back button.

This redirected to a **Not Found** page, showing that the value of `returnPath` was being used as the destination of the Back link.

## How did I verify it?

I inspected the Back link and observed that the value supplied through `returnPath` was placed inside the anchor's `href` attribute.

I then changed the parameter to:

`javascript:alert(document.cookie)`

The resulting URL was:

`https://0a9a0087046ea8f78020fd7f005200aa.web-security-academy.net/feedback?returnPath=javascript:alert(document.cookie)`

After loading the page and clicking the **Back** link, an alert displaying `document.cookie` appeared.

This confirmed the DOM XSS vulnerability.

## What happens?

The application takes the `returnPath` value from `location.search` and uses it as the `href` of the Back link.

Normally:

`returnPath=/`

results in a link similar to:

`<a href="/">Back</a>`

When the attacker supplies:

`javascript:alert(document.cookie)`

the link becomes conceptually:

`<a href="javascript:alert(document.cookie)">Back</a>`

When the victim clicks the link, the browser interprets the `javascript:` URL and executes:

`alert(document.cookie)`

The execution flow is:

`location.search` → `returnPath` → jQuery `href` assignment → `<a href="javascript:...">` → click → JavaScript execution

## Why does it work?

The vulnerability exists because attacker-controlled data from `location.search` is placed directly into an anchor's `href` attribute without safely restricting the URL scheme.

The important part is the use of:

`javascript:`

This is not treated as ordinary text in an `href`. It is a URL scheme that can cause JavaScript execution when the link is activated.

This is why the following works:

`javascript:alert(document.cookie)`

while HTML-based payloads such as:

`<script>alert(document.cookie)</script>`

do not work in this context.

The payload is being used as an **`href` value**, not inserted as a new HTML element.

## Important Observation

The `/abcd` test was useful for identifying the vulnerable behavior.

Using:

`returnPath=/abcd`

caused the Back link to navigate to a non-existent `/abcd` path.

This demonstrated that the `returnPath` parameter was controlling the destination of the Back link.

After confirming that behavior, the next step was to test whether a different URL scheme could be supplied:

`returnPath=javascript:alert(document.cookie)`

This changed the behavior from normal navigation to JavaScript execution.

## What can an attacker do?

If an attacker can control the `returnPath` value and convince another user to visit the malicious URL, the attacker may be able to execute JavaScript in the context of the vulnerable application when the victim clicks the Back link.

Depending on the application's security controls and the victim's privileges, this could potentially allow:

* Accessing sensitive data available to JavaScript
* Performing actions as the victim
* Manipulating page content
* Stealing information accessible to the script
* Chaining the XSS with other vulnerabilities

## What are the conditions/limitations?

* The attacker must be able to control the `returnPath` parameter.
* The application must place that value into the anchor's `href`.
* Dangerous URL schemes such as `javascript:` must not be blocked or sanitized.
* The malicious link must be visited by a victim.
* The victim must activate the vulnerable Back link for this particular payload.

## How should it be fixed?

* Do not place untrusted user input directly into an `href` attribute.
* Allow only expected URL formats and schemes, such as relative paths or `https:`.
* Explicitly reject dangerous schemes such as `javascript:`.
* Validate and normalize the destination before assigning it to `href`.
* Avoid using attacker-controlled URL parameters as navigation targets without validation.
* Use a strong **Content Security Policy (CSP)** as defense-in-depth.

## What did I learn?

* `location.search` can be a DOM XSS source.
* jQuery's `attr()` can become dangerous when assigning attacker-controlled values to sensitive attributes.
* The **sink/context determines the appropriate payload**.
* An `href` attribute is different from an HTML body or `innerHTML` context.
* `javascript:` can turn an attacker-controlled link into a JavaScript execution point.
* Testing with a random value such as `/abcd` is useful for confirming where input is being used.
* DOM XSS can occur without the server directly reflecting the payload into the response.

## What was the key obstacle?

The main challenge was understanding why HTML payloads such as `<script>` or `<img>` were not effective.

The key was identifying that the input was being placed into an **anchor's `href` attribute**. Therefore, the payload needed to be valid and dangerous within a URL context.

Using:

`javascript:alert(document.cookie)`

matched the vulnerable context and successfully triggered JavaScript execution when the Back link was clicked.
