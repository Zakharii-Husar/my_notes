
# XSS Summary: The "Impersonation" Attack

### The Initial Confusion

At first glance, Cross-Site Scripting (XSS) seems weak because:

1. The code only runs in a browser, not on the server.
2. If an attacker runs code in their *own* browser, it does nothing harmful.
3. It’s unclear how client-side code can affect the server's data.

### The "Aha!" Moment: It’s Not Hacking the Server, It’s Hacking the *User*

The danger isn't that the script runs; the danger is **where** it runs.

* **The Context:** When a script runs on `bank.com`, the browser gives that script full access to everything `bank.com` knows about the user (cookies, session tokens, private data).
* **The Bridge to the Server:** The script doesn't need to "hack" the server. It simply uses the victim's **already logged-in session** to send commands (like "transfer money" or "send me the password").
* **The Server's Perspective:** The server receives a request accompanied by valid cookies. It cannot distinguish between the user clicking a button and a malicious script sending a fetch request. It trusts the request because it trusts the user.

### How the Attacker "Attaches" the Script

The attacker needs to force the victim's browser to run the script. They do this by finding a "hole" where the website takes input and puts it on the screen without cleaning it first.

**1. The "Malicious Link" Method (Reflected XSS)**

* **Scenario:** A search bar reflects input back to the user (`You searched for: [input]`).
* **The Attack:** The attacker crafts a URL where the "search term" is actually JavaScript code.
* **The Delivery:** The attacker sends this link to the victim via email or chat.
* **The Execution:** When the victim clicks, *their* browser loads the site, sees the script in the URL, and executes it, thinking it's part of the website.

**2. The "Landmine" Method (Stored XSS)**

* **Scenario:** A comment section saves text to a database and displays it to others.
* **The Attack:** The attacker posts a comment containing the script.
* **The Delivery:** The server saves the script.
* **The Execution:** Any user who views that comment section automatically runs the script, as their browser downloads the comment from the database.

### The Payload: What happens next?

Once the script is running in the victim's browser, the attacker effectively **becomes** the victim.

* **Bypassing Protection:** Because the script runs from the *inside* (same origin), it bypasses CORS and reads CSRF tokens.
* **Session Hijacking:** The script sends `document.cookie` to the attacker, giving them the keys to the account.
