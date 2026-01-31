***

# Kits Kärneriks: Breaking the Glass on Client-Side Fraud Defense

## 1. The Setup

I spent the better part of 2021 tearing apart the invisible force field that guards Google’s login infrastructure. The target was "Botguard," a piece of defensive engineering that sits quietly in the browser and decides if you are a human or a script. This wasn't an academic exercise in parsing HTML; I wanted to know if I could forge the digital passport required to enter the ecosystem. The result was a proof-of-concept that bypassed the checks completely by outsourcing the heavy lifting to a hardened, headless browser. We proved that even the most sophisticated client-side validation crumbles when you control the environment it runs in.

> 🚨 **Updated Context & Disclaimer** 🚨
>
> This analysis was conducted in 2021 on Google's v2 login flow. While Google rolled out a v3 login page in 2022, this update was largely a "facelift."
>
> The fundamental flaw discussed here—namely, Botguard tokens not being tied to a browser session—persisted in the v3 flow. The changes were minor, such as renaming the request parameter used to send the token. Consequently, the core principles of this analysis remain relevant for understanding the vulnerability, even if specific implementation details have changed. This content is for educational and research purposes only.


## 2. Dissecting Botguard

#### Purpose: Anti-Fraud and Anti-Phishing, Not Just Anti-Bot

While its name implies a generic "anti-bot" function, Botguard's primary purpose is more specific: **anti-fraud**. It is not merely designed to stop web scraping but to thwart automated attacks that lead to account takeover (ATO), such as credential stuffing, password spraying, and, most relevant to this research, **validating credentials at scale from a phishing proxy**.

A successful MITM phishing tool like Evilginx must not only look like the target site but also behave like it from the server's perspective. When a user enters their credentials into the phishing page, the tool proxies that request to the real service. Botguard's role is to ensure the environment where the credentials were entered was a legitimate, non-automated, human-driven browser session, thus invalidating the proxied request from the phishing server.

#### How It Functions: Client-Side Environmental Fingerprinting

It works by executing a massive suite of obfuscated JavaScript that fingerprint the client. It measures everything: the specific rendering quirks of your fonts, the microscopic jitter in your mouse movements, the dimensions of your screen, and the timing of your keystrokes. It bundles this chaos into a fingerprint and mints a security token.

This data is processed through a proprietary algorithm to generate a security token. This token essentially serves as the browser's "attestation" that the session is legitimate. The token is then sent as a `bgRequest` parameter with the account lookup request. If the token is missing, malformed, or decodes to a fingerprint that flags the session as "high-risk" or "automated," Google's servers reject the login attempt with the generic "Couldn't sign you in" error, providing no information to the attacker.


#### Drawbacks and Limitations

The weakness here is obvious if you stop thinking like a defender. The system relies entirely on code executing on the client's machine. If I control the machine, I control the reality that the code perceives. The check is domain-specific, meaning it fails on a phishing site, but the token itself is just a string of characters. If I could generate that string on a legitimate domain and smuggle it out, the server wouldn't know the difference.

## 3. The Heist

I started by watching my own attacks fail. My Man-in-the-Middle (MITM) proxy was getting rejected every time it tried to forward a login request. I traced the failure to a single parameter: bgRequest. This was the token. My proxy couldn't generate a valid one because it wasn't google.com.

#### The Breakthrough: Isolating the `bgRequest` Token

Through methodical network analysis and request replay (using tools like Burp Suite), it was discovered that the single differentiating factor was the value of the `bgRequest` parameter. A token generated on the phishing domain was invalid, while one generated on `google.com` was valid.


This led to the core hypothesis: **the server-side validation is primarily concerned with the integrity of the submitted token, not the immediate origin of the request itself.**

#### The Proof-of-Concept: Decoupled Token Generation

The solution was to build a token factory. I didn't try to reverse-engineer the obfuscated JavaScript because that is a losing battle against a team of Google engineers. Instead, I simply gave the script exactly what it wanted. I used go-rod, a browser automation framework, to spin up a headless Chrome instance. Naked automation gets flagged instantly, so I wrapped it in stealth libraries to patch the webdriver leaks and mask the user agent.


This "puppet" browser navigated to the real accounts.google.com. To Botguard, this looked like a legitimate user on a legitimate site. The script ran, did its biometric checks, and minted a pristine, valid token. I set up an interceptor to snatch that token the millisecond it was generated, killed the network request so it wouldn't be used, and exported the string.


I then fed this stolen ID card into my phishing proxy. When the proxy sent the login request to Google, it attached the valid token I had just manufactured in the clean environment. The server validated the token, saw it was legitimate, and authorized the session. We bypassed the entire fraud detection suite by simply decoupling the generation of the token from its usage.


<table>
  <tr>
    <td><img width="1024" height="512" alt="botguard_white" src="https://github.com/user-attachments/assets/b2304d98-c1a0-4d9b-8c10-f62aceb2d1fe" /></td>
    <td><img width="1024" height="512" alt="botguard_dark" src="https://github.com/user-attachments/assets/c8515399-ef0f-4260-9e4b-998a8ff31104" /></td>
  </tr>
</table>

## 4. The Reality Check

*   **Client-Side Checks are Vulnerable to Environmental Spoofing:** While powerful, client-side defenses are fundamentally running on an attacker-controlled machine. With sufficient effort, the environment can be mimicked to satisfy the checks. The use of stealth plugins is a clear example of this.
*   **Token Portability is a Potential Weakness:** The security of token-based systems like Botguard relies on the token being non-transferable. This research shows that if token generation can be outsourced to a "clean" environment, the token can then be used in a "dirty" one, defeating the purpose of the check.
*   **The Importance of a Layered Defense:** This bypass works because it circumvents a single, albeit strong, layer of defense. More advanced defensive systems could correlate the token's fingerprint with other server-side signals (e.g., IP address reputation, historical session data) to detect this type of anomaly.

The industry creates these elaborate cat-and-mouse games, adding layers of obfuscation and behavioral analysis. Yet, as long as the logic relies on a token that can be ported from a clean environment to a dirty one, the defense is bypassable. It isn't about breaking the encryption; it is about abusing the architecture.

## 5. Contact

I break things to understand how to build them better. I am looking for roles where I can apply this offensive mindset to defensive strategies. If you want to discuss web security, reverse engineering, or why client-side trust is a myth, reach out. Find me on LinkedIn.