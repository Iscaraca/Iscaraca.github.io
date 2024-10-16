---
layout: post
title: Log File Poisoning from Local File Inclusion
author: Isaac
redirect_from: /2022-09-05-log-poisoning
---

{% raw %}
## Local File Inclusion

LFIs allow users to read files on the server that are not meant to be accessed, typically as a result of poor input validation. The page to be displayed can be accessed in a multitude of ways, some of which are detailed below:

<br>

| Type | Example |
| --- | --- |
| Url parameter | `http://example.com/index.php?file=PATH_TO_PAGE` |
| Cookie | `PHPSESSID=O:9:"PageModel":1:{s:4:"file";s:15:"PATH_TO_PAGE";}` |
| POST parameter | `file=PATH_TO_PAGE` |

<br>

Normal usage of the site would allow users to navigate between pages meant for public viewing, but if the page path is not properly validated, changing this value on the client side would allow users to view other pages on the server.

<br>

While this vulnerability allows us to view sensitive data that may be hidden from normal users, we can go a step further to carry out Remote Code Execution on the server using PHP, even if there aren't any obvious user input attack vectors we can use to plant PHP payloads on the site.

<br>

Useful information about LFIs for PHP can be found in these sites:
- [General PHP LFI information](https://ctf--wiki-org.translate.goog/en/web/php/php/?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=zh-CN).
- [Exploiting temporary file uploads](https://rafalharazinski.gitbook.io/security/other-web-vulnerabilities/local-remote-file-inclusion/phpinfo-log-race-condition)

<br>

## Log File Poisoning

If the particular LFI vulnerability allows us to access the log files on the server, we can often pass arbitrary user input by creating logs that include PHP payloads, and then navigating to those log files.

<br>

Take, for example, a website that handles the user's current state using the PHPSESSID cookie `Tzo5OiJQYWdlTW9kZWwiOjE6e3M6NDoiZmlsZSI7czoxNToiL3d3dy9pbmRleC5odG1sIjt9` which decodes to `O:9:"PageModel":1:{s:4:"file";s:15:"/www/index.html";}`. By changing this value to access the `/etc/passwd` file on a linux server, `O:9:"PageModel":1:{s:4:"file";s:23:"/../../../../etc/passwd";}`, we can see that the cookie is vulnerable to LFI:

![passwd file](/assets/img/log-poisoning/lfi-passwd.png)

<br>

There are multiple log files that can exist on a linux server for different purposes. Common log files are listed [here](https://www.cyberciti.biz/faq/linux-log-files-location-and-how-do-i-view-logs-files/). In the case of the website above, we see that there are database entries for an nginx web server, so it would be worth checking out the nginx logs at `/var/log/nginx/`. For nginx, there are two notable files, `access.log` and `error.log`. Let's check `access.log` out.

![access log](/assets/img/log-poisoning/access-log.png)

<br>

We see that `access.log` stores the HTTP requests sent to the server. If we send a HTTP request to any path on the server, even if the path is nonexistent, our request will be displayed on the log. A notable feature of PHP is that any inclusions will execute the contents of the file as PHP code. We can thus take advantage of this by sending a HTTP request, replacing any logged content with a PHP payload, such as `<?php system($_GET['c']); ?>`. Navigating to the log file again will reveal a new log entry with our executed payload.

{% endraw %}