***

# Kits Kärneriks: A Case Study in Bypassing Client-Side Anti-Fraud Mechanisms

## 1. Executive Summary

This repository documents a 2021 research project into the client-side validation mechanisms of Google's login infrastructure. The primary focus is the analysis of "Botguard," a sophisticated JavaScript-based system designed for anti-fraud and anti-phishing purposes.

The objective was to investigate whether Botguard's defenses, which are deeply embedded in the client-side browser environment, could be circumvented. The research successfully developed a proof-of-concept method that involved using a hardened headless browser to generate a valid security token on the legitimate Google domain, which was then used to authorize a session in a separate Man-in-the-Middle (MITM) context.

This project serves as a case study on the strengths and limitations of client-side security tokens and demonstrates the perpetual "cat-and-mouse" game between platform defenders and security researchers.

> 🚨 **Updated Context & Disclaimer** 🚨
>
> This analysis was conducted in 2021 on Google's v2 login flow. While Google rolled out a v3 login page in 2022, this update was largely a "facelift."
>
> The fundamental flaw discussed here—namely, Botguard tokens not being tied to a browser session—persisted in the v3 flow. The changes were minor, such as renaming the request parameter used to send the token. Consequently, the core principles of this analysis remain relevant for understanding the vulnerability, even if specific implementation details have changed. This content is for educational and research purposes only.

## 2. Understanding Google's Botguard

Before detailing the methodology, it is crucial to understand what Botguard is and, more importantly, what it is designed to prevent.

#### Purpose: Anti-Fraud and Anti-Phishing, Not Just Anti-Bot

While its name implies a generic "anti-bot" function, Botguard's primary purpose is more specific: **anti-fraud**. It is not merely designed to stop web scraping but to thwart automated attacks that lead to account takeover (ATO), such as credential stuffing, password spraying, and, most relevant to this research, **validating credentials at scale from a phishing proxy**.

A successful MITM phishing tool like Evilginx must not only look like the target site but also behave like it from the server's perspective. When a user enters their credentials into the phishing page, the tool proxies that request to the real service. Botguard's role is to ensure the environment where the credentials were entered was a legitimate, non-automated, human-driven browser session, thus invalidating the proxied request from the phishing server.

#### How It Functions: Client-Side Environmental Fingerprinting

Botguard operates as a complex and heavily obfuscated JavaScript suite that executes on the client-side (the user's browser) during the login process. Its core function is to generate a highly detailed fingerprint of the browser environment and user behavior. It collects hundreds of data points, which may include:

*   **Browser/DOM Properties:** `navigator` properties, screen resolution, installed fonts, browser plugins, and specific DOM element timings.
*   **Behavioral Biometrics:** Timing of keystrokes, mouse movement patterns, and interaction speed.
*   **Environment Integrity:** Checks for signs of automation frameworks (like `webdriver` flags), debugging tools, or inconsistencies that suggest a non-standard browser.

This data is processed through a proprietary algorithm to generate a security token. This token essentially serves as the browser's "attestation" that the session is legitimate. The token is then sent as a `bgRequest` parameter with the account lookup request. If the token is missing, malformed, or decodes to a fingerprint that flags the session as "high-risk" or "automated," Google's servers reject the login attempt with the generic "Couldn't sign you in" error, providing no information to the attacker.

#### Drawbacks and Limitations

No system is perfect. Botguard's reliance on client-side validation presents inherent challenges:

1.  **The Arms Race:** Its effectiveness depends on its detection logic remaining a secret. Once researchers or attackers identify the signals it checks for, they can begin to spoof them. This leads to a constant cycle of updating the JS and the evasion techniques.
2.  **Environmental Brittleness:** The core problem this research addresses is that the check is **domain-specific**. The JavaScript is designed to run on `accounts.google.com`. When executed on a different domain (the phishing page), even if the JS code is identical, environmental checks (like `window.location.hostname`) fail, leading to an invalid token.
3.  **Transferability of the Token:** Because the validation is encapsulated within a generated token, the system's security hinges on the assumption that a valid token cannot be generated outside of a legitimate context. If an attacker can find a way to generate a valid token *somewhere* and then use it *elsewhere*, the check is effectively bypassed.

## 3. Research Methodology & Proof-of-Concept

The initial analysis began by observing failing login attempts from a standard MITM proxy. Requests proxied from the phishing domain were consistently rejected by Google's servers, while identical requests initiated from the legitimate Google domain succeeded.

#### The Breakthrough: Isolating the `bgRequest` Token

Through methodical network analysis and request replay (using tools like Burp Suite), it was discovered that the single differentiating factor was the value of the `bgRequest` parameter. A token generated on the phishing domain was invalid, while one generated on `google.com` was valid.

This led to the core hypothesis: **the server-side validation is primarily concerned with the integrity of the submitted token, not the immediate origin of the request itself.**

#### The Proof-of-Concept: Decoupled Token Generation

To prove this, a system was engineered to decouple the token generation from the MITM session. The process was as follows:

1.  **Automate a Legitimate Environment:** A headless browser framework, **[go-rod](https://github.com/go-rod/rod)**, was used to automate a browser session. Crucially, this session navigated directly to the real `accounts.google.com`.
2.  **Evade Detection:** Headless browsers are easily detectable. To circumvent this, the **[go-rod/stealth](https://github.com/go-rod/stealth)** package was employed. This package patches the browser automation framework to remove common detection vectors (e.g., hiding the `webdriver` flag, mimicking a standard user agent, and correcting other environmental inconsistencies).
3.  **Generate and Intercept the Token:** The automated browser would enter the target's email address on the legitimate Google page. This action triggers the Botguard JavaScript to generate a valid token. The outgoing `/accountLookup` request was then intercepted *on the client-side*, and the valid `bgRequest` token was extracted from its parameters. The request itself was blocked to prevent it from reaching Google's servers, avoiding rate-limiting or other flags.
4.  **Inject and Execute:** This valid, freshly generated token was then passed to the MITM tool, which injected it into the phishing session's `/accountLookup` request. When this request was proxied to Google's servers, it was accepted, and the login flow proceeded to the password entry stage.

To improve performance for this proof-of-concept, the process was wrapped in a simple REST API, allowing the MITM tool to request a fresh token on-demand.

<table>
  <tr>
    <td><img width="1024" height="512" alt="botguard_white" src="https://github.com/user-attachments/assets/b2304d98-c1a0-4d9b-8c10-f62aceb2d1fe" /></td>
    <td><img width="1024" height="512" alt="botguard_dark" src="https://github.com/user-attachments/assets/c8515399-ef0f-4260-9e4b-998a8ff31104" /></td>
  </tr>
</table>

## 4. Key Findings and Implications for Web Security

This research demonstrated a practical method for bypassing a sophisticated, client-side, anti-fraud system by exploiting its architectural limitations. The key takeaways are:

*   **Client-Side Checks are Vulnerable to Environmental Spoofing:** While powerful, client-side defenses are fundamentally running on an attacker-controlled machine. With sufficient effort, the environment can be mimicked to satisfy the checks. The use of stealth plugins is a clear example of this.
*   **Token Portability is a Potential Weakness:** The security of token-based systems like Botguard relies on the token being non-transferable. This research shows that if token generation can be outsourced to a "clean" environment, the token can then be used in a "dirty" one, defeating the purpose of the check.
*   **The Importance of a Layered Defense:** This bypass works because it circumvents a single, albeit strong, layer of defense. More advanced defensive systems could correlate the token's fingerprint with other server-side signals (e.g., IP address reputation, historical session data) to detect this type of anomaly.

The ideal solution from an attacker's perspective would be a complete reverse-engineering of the obfuscated JavaScript to generate valid tokens without browser automation. However, this proof-of-concept illustrates that such a time-intensive effort is not always necessary if an architectural workaround can be found.

## 5. Contact

As a security researcher and software developer, I am passionate about understanding and improving web security. I am actively seeking opportunities to apply my skills in a defensive or research-oriented role. I welcome discussions on web application security, reverse engineering, and defensive strategies. Please connect with me on LinkedIn or WhatsApp.
