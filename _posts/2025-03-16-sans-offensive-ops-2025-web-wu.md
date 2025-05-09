---
layout: post
title: Web CTF Writeup of SANS Offensive Operations 2025
date: 2025-03-19
permalink: "/blogs/sans-offensive-ops-2025-web-wu.html"
---

Recently (at the time of this writing), me and my friends/colleagues participated in SANS Offensive Operations CTF 2025. A various jeopardy-style CTF that includes network pentesting, web applications, as well as binary exploitation, I managed to finish five web application-based challenges that I can solved, ended up in the 48th position.

![image](/assets/sans-offensive-ops-25/20250319141510.png)

# Table of contents
- [Ghibli Shop (API Security Testing)](#ghibli-shop-api-security-testing)
- [Muffin Papers (SQL Injection)](#muffin-papers-sql-injection)
- [Ecorp Tools (Web Attacks)](#ecorp-tools-web-attacks)

---

# [Ghibli Shop (API Security Testing)](#ghibli-shop-api-security-testing)

## 001

> We're working on a new Ghibli Shop to sell the cutest plush toys you'll find on the internet! We don't have all the products in stock yet, can you find the hidden item?

> Note: No fuzzing or bruteforce is required. Stick to application logic.

### Web page identification

Accessing the web page
![image](/assets/sans-offensive-ops-25/20250318220830.png)

There are eight products shown in the web application. Hovering the endpoints will get the products only (within the format `ghibli.pwn.site:8035/products/n`)

By accessing each products, the web will send the API GET request to obtain the product with the `id=2`
![image](/assets/sans-offensive-ops-25/20250318221216.png)
![image](/assets/sans-offensive-ops-25/20250318221430.png)

Looking back at the *Target* tab in my Burp Suite, I examined each tab of endpoints, which requires me to click each products provided in the web page.
![image](/assets/sans-offensive-ops-25/20250318221045.png)

The endpoint `/api/products` seems interesting, so let's check it out by copying the request to the *Repeater* tab.
![image](/assets/sans-offensive-ops-25/20250318221654.png)

The API sends the POST request of the existing products through the parameter `"instock":true`. If the value is `true`, the API will display the entire available products.

Otherwise, if I tried to change the value into `"instock":false`
![image](/assets/sans-offensive-ops-25/20250318221834.png)

There's a hidden product which also serves the first flag of Ghibli Shop.

# [Muffin Papers (SQL Injection)](#muffin-papers-sql-injection)

## 001

> We are tasked with pentesting the Dunder Muffin company's upcoming E-commerce website. They value their customer's privacy and have asked us to perform a blackbox assessment. Please take a look and see if you can find any vulnerability that can get you a flag.

> Note: No fuzzing or bruteforce is required. Stick to application logic.

### Web page identification

![image](/assets/sans-offensive-ops-25/20250318233142.png)

Hovering the sites will display the only accessible endpoint directory, which is `papers.pwn.site:8015/shop`

![image](/assets/sans-offensive-ops-25/20250318233210.png)

Testing for the search query by searching `muffin`
![image](/assets/sans-offensive-ops-25/20250318233248.png)

I can detect that this is possibly the SQL injection vulnerability, but we need to find out how many columns exist in the table.

Starting with `' order by 10-- -` until `'order by 1-- -`
![image](/assets/sans-offensive-ops-25/20250318233409.png)

No response shown within the web page but the error message that says `Your query didn't yield any results!`.

I noticed there's a checkbox `Paper`. We can try to repeat the search again what does the checkbox do.
![image](/assets/sans-offensive-ops-25/20250318233517.png)

It seems that the checkbox adds a parameter `filter` in the URL.

Let's try for the injection against the new parameter, within the same method (`' order by 10-- -` until `'order by 1-- -` against the parameter `filter`)
![image](/assets/sans-offensive-ops-25/20250318233650.png)

I found there are five columns in the table, which is good.

### SQL injection to retrieve the flag

Defining the inserted values by specifying the SQL version
```
paper' UNION select NULL,NULL,<version>,NULL,NULL-- -
```
by trying possible versions that I can use
- MySQL: `@@version`
- PostgreSQL: `version()`

![image](/assets/sans-offensive-ops-25/20250318234200.png)

I found the web app is using PostgreSQL as their database.

Figuring out the existing databases, I use the following payload against the `filter` parameter URL
```
paper' UNION select NULL,NULL,schema_name,NULL,NULL from INFORMATION_SCHEMA.SCHEMATA-- -
```
![image](/assets/sans-offensive-ops-25/20250318234348.png)

There are four databases exist
- `PG_TOAST`
- `PG_CATALOG`
- `INFORMATION_SCHEMA`
- `PUBLIC`

Since the first three databases are set by default from Postgre, I chose the `PUBLIC` database, because I thought that the table entries are set in the `PUBLIC` database.

Moving on, I'd like to know about the tables in the database `PUBLIC`, so I continued with the following payload.
```
paper' UNION select NULL,NULL,table_name,NULL,NULL from INFORMATION_SCHEMA.TABLES where table_schema='public'-- -
```
![image](/assets/sans-offensive-ops-25/20250318234933.png)

There are two tables shown, `PAPERS` and `USERS`. I might assume the values of the table `PAPERS` are the list of paper products, which has been shown in the `Special Deals` section, as well as the search results. 

Next, I'd like to know the entire table values in the table `USERS`, so I continued with the following payload
```
paper' UNION select *,NULL,NULL from public.users-- -
```
- Note that I removed the first two `NULL`s, due to the fact that it returns an error
![image](/assets/sans-offensive-ops-25/20250318235434.png)

Finally obtained the first flag of Muffin Papers.

## 002

> Can you get code execution on the server? The flag is in the `/` directory.
>
> Note: No fuzzing or bruteforce is required. Stick to application logic.

### SQL injection to read files

Continuing the SQL injection, the intended answer is to read the flag file in the `/` (root) directory.

I read on the PostgreSQL file manipulation from [Payloads All The Things repository](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/PostgreSQL%20Injection.md#postgresql-file-read), which according from the information:

> Earlier versions of Postgres did not accept absolute paths in `pg_read_file` or `pg_ls_dir`. Newer versions (as of [0fdc8495bff02684142a44ab3bc5b18a8ca1863a](https://github.com/postgres/postgres/commit/0fdc8495bff02684142a44ab3bc5b18a8ca1863a) commit) will allow reading any file/filepath for super users or users in the `default_role_read_server_files` group.

Using the same payload position, I tried to enumerate the root directory by using the payload `pg_ls_dir('/')`, specifying the file enumeration on the root directory. The URL payload should look like this
```
paper' UNION select NULL,NULL,pg_ls_dir('/'),NULL,NULL-- -
```
![image](/assets/sans-offensive-ops-25/20250319000401.png)

Found that the flag is under the `/flag_e5c2fc41110b168.txt`. Finally, we can use the `pg_read_file` function of the flag file location against the SQL payload.
```
paper' UNION select NULL,NULL,pg_read_file('/flag_e5c2fc41110b168.txt', 0, 200),NULL,NULL-- -
```
![image](/assets/sans-offensive-ops-25/20250319000605.png)

Obtained the final flag for Muffin Papers.

# [Ecorp Tools (Web Attacks)](#ecorp-tools-web-attacks)
---
## 001

> There's not much time to explain, we must get access to the ecorp backup server before the Dark Army does. We've managed to get a foothold on the on the network, the backup server is hosting an internal tools repository. Can you find a way to get access to the server?

> Task: Read the contents of the `/opt/flag.txt` file on the server.

> Note: No fuzzing or bruteforce is required. Stick to application logic.

### Web page identification

![image](/assets/sans-offensive-ops-25/20250319002429.png)

I found there are four internal tools that can be used. The given question is to read `/opt/flag.txt` on the web application. First, let's try for the `XML to JSON` tool, which might gives us an idea to perform XXE.

### XML External Entity to read sensitive files

Examining the tool `XML to JSON`, we found there is a function to convert XML texts to JSON
![image](/assets/sans-offensive-ops-25/20250319003819.png)
![image](/assets/sans-offensive-ops-25/20250319003828.png)

Look out for the XML and JSON and see the correlations of the value and how it reflected back. The `<Location/>` value on the XML is reflected back to the `"Location":` in JSON.

I use  XML `Location` parameter as the XML point to display the output of XXE attack, which expects us to be able to read the sensitive file (`/opt/flag.txt`).

To craft the XXE payload, add the `<!DOCTYPE>`, that includes `<!ENTITY>` as the XML [Document Type Definition (DTD)](https://www.w3schools.com/xml/xml_dtd.asp). Next, the `<!ENTITY>` serves as the external entity, which gives us the ability to define the contents of a file. Finally, we can serve the file with `SYSTEM file:///`, which the `file:///` serves as the `/` root directory.

Within that information, let's craft our DTD from that scenario.
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE ServerDetails [
<!ENTITY statenisland SYSTEM "file:///opt/flag.txt">
]>
```

The `ENTITY` `statenisland` will be declared in the XML as the external entity, which will be declared inside of the XML parameter `<Location/>`
```
<EcorpBackupServer>
	<ServerDetails>
		...
		<Location>&statenisland;</Location>
		...
```

![image](/assets/sans-offensive-ops-25/20250319004759.png)
![image](/assets/sans-offensive-ops-25/20250319004811.png)

Found the first flag of Ecorp Tools.

## 002

> Since we can read the server files now, let's retrieve the application source code.

> Task: Locate the application's index controller file to discover hidden application endpoints.

> Note: No fuzzing or bruteforce is required. Stick to application logic.

### XML external entity to read source code

Using the same payload, we need to find the source code of the web application, which only changing the `file://` path.

By default, the web root directory (as in Linux) is located under the `/var/www`, so we try to change the previous SYSTEM keyword to `file:///var/www`
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE ServerDetails [
<!ENTITY statenisland SYSTEM "file:///var/www">
]>
...
```
![image](/assets/sans-offensive-ops-25/20250319010746.png)

Found several files and directories
- `.gitignore`
- `.m2`
- `/files`
- `pom.xml`
- `/src`
- `/target`
The `pom.xml` might contain interesting information, so let's check that out.
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE ServerDetails [
<!ENTITY statenisland SYSTEM "file:///var/www/pom.xml">
]>
```
![image](/assets/sans-offensive-ops-25/20250319011147.png)![image](/assets/sans-offensive-ops-25/20250319011203.png)

I found the current application is running Spring Boot, along with the templates and parsers, such as Thymeleaf, SnakeYAML, and more.

During the research, I found the Spring Boot project guide, where I can examine their web root structure, in expectation to obtain the `index` page. According from the [AI Bot Assistant from Quora](https://www.quora.com/Where-should-I-keep-an-index-HTML-file-in-a-spring-boot-project),
	![image](/assets/sans-offensive-ops-25/20250319012525.png)
Within this structure, I can find the `index` by following the structure above. First, I tried to run to the `/static` directory, which there's no `index.html` found. Instead, I went for the section `yourpackage`, where I can find several information of Java-based file that might serve HTML and web application scripts. 
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE ServerDetails [
<!ENTITY statenisland SYSTEM "file:///var/www/src/main/java/com/ecorp/ecorp_tools">
]>
```
![image](/assets/sans-offensive-ops-25/20250319014224.png)

There are three directories and a file found, which are shown as:
- `/config`
- `/controller`
- `EcorpToolsApplication.java`
- `/utils`

Going further, I found that the Java's web application is often defined in `controller.IndexController.java` class, which serves the main purpose as the HTML page in most common web applications.

For example, when a user visits `http://yourwebsite.com/`, this `controller.IndexController.java` might handle the request as in the following
```
@Controller
public class IndexController {

    @GetMapping("/")
    public String index() {
        return "index"; // Renders the index.html or index.jsp view
    }
}
```
In this case, it returns the name of a view (e.g., `index.html`, `index.jsp`, or `index.thymeleaf`) that represents the homepage of the application.

Checking out the `/controller/IndexController.java`

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE ServerDetails [
<!ENTITY statenisland SYSTEM "file:///var/www/src/main/java/com/ecorp/ecorp_tools/controller/IndexController.java">
]>
```
![image](/assets/sans-offensive-ops-25/20250319014407.png)

Found an application's main source code, as well as a second flag for Ecorp Tools.