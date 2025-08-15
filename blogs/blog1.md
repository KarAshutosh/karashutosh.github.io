
# Reflected XSS Filter Bypass in Search Functionality

Two years ago, while testing a web application, I discovered an interesting **Reflected Cross-Site Scripting (XSS)** vulnerability.
The site had some filtering in place, but with a bit of creative input crafting, it was possible to bypass the protection and execute arbitrary JavaScript in a victim’s browser.

The vulnerability has since been **fixed**, but I’m sharing the details here for educational purposes and to highlight a common sanitization oversight.

---

## What is Reflected XSS?

Reflected XSS occurs when user-supplied data is immediately included in a server response without proper sanitization or encoding.
Attackers can inject malicious JavaScript into a URL, tricking a victim into clicking it.
When the victim’s browser loads the page, the script runs with the same permissions as the legitimate site.

---

## Target:

`https://exceleratorparts.com/dtna/en/`

The vulnerable parameter was found in the **search** functionality.

---

## The Filter Problem

At the time, the site’s filter removed HTML tags enclosed in `< >`.
However, the filter made a critical assumption: it only stripped tags if both the **opening `<` and closing `>`** were present.

This meant that if the closing `>` was omitted and replaced with `/<`, the filter’s rule could be bypassed while the browser still interpreted the tag.

---

## Crafting the Payload

I chose to inject an `<input>` element because:

* It can have focus-related events like `onfocus` to trigger JavaScript.
* It can be made to auto-focus when the page loads, ensuring the payload executes without user interaction.
* It blends into the page without raising suspicion.

Here’s the payload:

```html
<input type="text" id="search-text" name="query" value="" onfocus="alert(1)" autofocus="" /<
```

### Attribute Breakdown

* `type="text"` → Standard text input for visibility and interactivity.
* `id="search-text"` → Matches existing DOM patterns for consistency.
* `name="query"` → Matches the backend parameter name used in searches.
* `value=""` → Keeps the field empty.
* `onfocus="alert(1)"` → Proof-of-concept payload (can be replaced with malicious JavaScript).
* `autofocus=""` → Automatically focuses the input, triggering the payload immediately.
* `/ <` → Filter bypass — prevents the filter from removing the tag while still being parsed by the browser.

---

## Proof of Concept (PoC)

Just visiting the following URL triggered the payload at the time:

```
https://exceleratorparts.com/dtna/en/search/?text=%3Cinput+type%3D%22text%22+id%3D%22search-text%22+name%3D%22query%22+value%3D%22%22+onfocus%3D%22alert%281%29%22+autofocus%3D%22%22+%2F%3C
```

When the page loaded, the injected input field automatically gained focus and triggered `alert(1)`.

---

## Disclosure Timeline

* **July 10, 2023** — Reported the vulnerability on HackerOne.
* The issue was **acknowledged** but marked as a **duplicate**, indicating it had already been reported by another researcher.
* The vulnerability has since been **patched and is no longer exploitable**.

---

## Impact (Before Fix)

While the proof-of-concept only showed a harmless alert, an attacker could have:

* Stolen session cookies (`document.cookie`) to impersonate users.
* Redirected users to malicious websites.
* Performed actions as the victim within the application.
* Loaded additional malicious scripts from external servers.

For authenticated sessions, this could have meant full account takeover.

---

## Severity

**Medium:** Any reflected XSS that can run arbitrary JavaScript poses a major risk. However, the need for a user to click on a malicious link reduces the overall severity.

---

## Recommendations

1. **Properly sanitize and encode** all user input before rendering it in HTML.
2. **Reject incomplete HTML tags** such as `<` without `>`.
3. **Use libraries** that auto-escape output (e.g., template engines with built-in escaping).
4. **Implement a strong Content Security Policy (CSP)** to mitigate script injection.

---

## Final Thoughts

This vulnerability is a textbook example of why **partial sanitization isn’t enough**.
Even if you remove dangerous tags, creative payload construction can bypass naive filters.

In this case, overlooking incomplete HTML tags allowed full XSS exploitation.
Security controls must be designed with **browser parsing behavior** in mind — not just what the filter “expects.”

A **simple HTML encoding step** (converting `<` to `&lt;`, `>` to `&gt;`, and so on) would have completely prevented this attack in the first place.
