# Breaking into Micro-CMS v1

## Flag 0: Insecure Direct Object Reference (IDOR)
Fuck, I used so many hints for this, even though it was actually way easier lol. This hint helped: "If the front door doesn't open, try the window".
 
Create two new pages. Spot that the website indexes this page sequentially:
```
<li>
    ::marker
    <a href="page/7">newpage1</a>
</li>
<li>
    ::marker
    <a href="page/8">newpage2</a>
</li>

```
Spot, that the very first post created by the admin is ```page/3```. So there is a lot missing between 3 - 7. And if you skip to page/4 you enter a page that is FORBIDDEN. We found the page we want to crack.
Using Network-Tab we see that the page only holds some Google-Analytics-Cookies. So no cookie hijacking probably.

The website has an option to edit webpages using ```page/edit/pagenumber```. Bingo:
```
page/edit/4
```
The edit endpoint doesn't check permissions.

## Flag 1:
XSS doesnt seems to work.
Path Traversal doesn't seem to work.
So SQL Injection it is. 

For the edit function to work the website has to store the previous contents of a file somewhere in a database. It pulls it with `/page/edit/4`. So lets see if this is sanitized:

```
https://f8ba173366a9388a84e5682ab9900a12.ctf.hacker101.com/page/edit/8'OR 1'=1 --
```
OR Operator evaluates Booleans. And 1 = 1 returns TRUE. The -- comments out the rest of the developer's query. So the POST to the database essentially becomes:
```
SELECT * FROM pages WHERE id = '8' OR '1'='1';
SELECT * FROM pages WHERE TRUE;
```
This lists all rows of the TABLE. So we see the entire database.
Flag found.


## Flag 2: Reflected XSS
/POST is unsanitized:
```
<script>alert(1+1)</script>
```
This could be used to ls the files in the server and then search
through them, possibly hitting admin privileges and then accessing


## Flag 3: Stored XSS
Using Markdown I can create a button on the website by default. `<button>click</button>`. I can use this to create a button with the event handler: 
```
<button onclick="javascript:alert(1)">click</button>
```
It creates a button. And if I click this (in developer tools) it shows me the flag.

## Discovery: Obfuscation By HTML-Encoding 
Test if the markdown sanitizer works correctly.

Figured out that the sanitizer just works something like this:
```
text = <a href=javascript:alert(1)></a>
if (text.includes === "script"){
    text.replace(/script/g, "scrubber");
}
```
The string `script` becomes `scrubbed`. So let's try and circumvent this parsing :). 

Figure out if the parser ignores backspaces or concatenation.
```
<scr\ipt>

<scr"+"ipt>
```
It failed.

But okay so the website parses it to markdown and then posts it. But after the conversion to markdown, the markdown is parsed to the website. And HTML decodes entities like `&#115;`(s) in attributes<a>.
```
<a href="java&#115;cript:alert(1)">Click</a>

```
Holy Guacamole, I can actually simplify this. Markdown sets <a href=> by itself, when you write:
```
[text](link)
[adorable kitten](java&#115;cript:alert(1)">Click</a>)
```
So in the `link` I can put in the `&#115`. Once Markdown converts it, it becomes:
```
<a href="java&#115cript:alert(1)">markdownvulnerability</a>
```
This is then handed to the website, which decodes the `&#115;` into s, thus it becomes `javascript` and the sanitisation is circumvented. It works! Doesn't give me a flag though lul.
