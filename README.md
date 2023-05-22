# Introduction to macOS - TCC

Continuing with interesting security measures on macOS - here comes TCC.  
`TCC (Transparency, Consent and Control)` is a macOS mechanism aimed at protecting sensitive information.  
This includes access to user's private files (e.g. files on the Desktop), access to the camera and the microphone, location services access and many more.  
Interestingly, TCC protects those even against root-level attacks.

## The user experience aspect
Remember the Windoews Vista days? One of the things that really impacted the user experience was the introduction of [UAC](https://learn.microsoft.com/en-us/windows/security/identity-protection/user-account-control/how-user-account-control-works).  
Microsoft has concluded around that time that many attacks are run by administrator-level access, which essentially meant they could do anything.  
UAC was supposed to take care of that problem by splitting the administrator token to two parts, and creating UAC "prompts" when there's a need to run a privileged operation.  
The user experience for that was not great - approving every little operation began to feel like one of those idle clicker games, and so UAC was changed to have 4 different levels of enforcement - this is what we experience today in Windows 10\11.  
This isn't a UAC talk (perhaps I'll do one in the future!). Apple decided to do something similar - prompting the user when some application is going to access private data. However, Apple had a slightly different approach, as they decided to *persistently keep the user's choice*.  
That's excellent from a user-experience perspective, since the user has to approve only *once*, and then that choice is used forever (or until the user decides to reset their choice manually). However, that means that the persistence of user's choice is the obvious target for attackers, and that's what we'll be examining today!

## Experimenting with TCC
In macOS, you'll normally see two instances of a daemon called `tccd` - one runs as root with the "system" argument, and one runs per logged-in-user:

```shell
jbo@McJbo ~ % ps aux | grep tccd | grep -v grep
jbo                529   0.0  0.0 33968064   8344   ??  S    Tue03PM   0:05.77 /System/Library/PrivateFrameworks/TCC.framework/Support/tccd
root               181   0.0  0.0 33969492  10284   ??  Ss   Tue03PM   0:14.37 /System/Library/PrivateFrameworks/TCC.framework/Support/tccd system
jbo@McJbo ~ %
```

Those daemons enforce TCC access on macOS - when a privileged operation is about to occur, the "correct" `tccd` instance is consulted by the operating system and it decides whether to approve or deny the request.  
When `tccd` gets a request, it will examine the "responsible App" alongside the type of request (e.g. access to camera) and consult a special database called `TCC.db`. If there is a record of that App with the requested type of operation - `tccd` will give its verdict based on the database record (yay\nay); otherwise, it will present the usual UI asking whether to approve or deny the request, and based on the user's choice will update the database.  
The system-wide database is located in `/Library/Application Support/com.apple.TCC/TCC.db` and the per-user one in `~//Library/Application Support/com.apple.TCC/TCC.db`. Let's examine one of those databases!

```shell
jbo@McJbo ~ % sudo file "/Library/Application Support/com.apple.TCC/TCC.db"
/Library/Application Support/com.apple.TCC/TCC.db: cannot open: Operation not permitted
jbo@McJbo ~ % 
```

Oh... Note that even though I ran as root, I still get an "Operation not permitted" error, even though it's just for reading!  
This happens because *TCC protects the Database files* - even viewing them is considered a sensitive operation!  
To resolve the issue, we could use the UI and assign `Full Disk Access` to `Terminal` ("System Settings" --> "Privacy & Security" --> "Full Disk Access").  
The "Privacy & Security" pane reflects the current state of the `TCC.db` instances and can reset/modify existing entries.  
With Full Disk Access, we're able to view the file:

```shell
jbo@McJbo ~ % file "/Library/Application Support/com.apple.TCC/TCC.db"
/Library/Application Support/com.apple.TCC/TCC.db: SQLite 3.x database, last written using SQLite version 3039005, file counter 256, database pages 14, cookie 0x49, schema 4, UTF-8, version-valid-for 256
jbo@McJbo ~ %
```

Note how we didn't even need to try and run as root. Since it's a SQLite database, we could simply use the `sqlite3` utility to read the database:

```shell
jbo@McJbo ~ % sqlite3 "/Library/Application Support/com.apple.TCC/TCC.db"
SQLite version 3.36.0 2021-06-18 18:36:39
Enter ".help" for usage hints.
sqlite> .dump access
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE access (    service        TEXT        NOT NULL,     client         TEXT        NOT NULL,     client_type    INTEGER     NOT NULL,     auth_value     INTEGER     NOT NULL,     auth_reason    INTEGER     NOT NULL,     auth_version   INTEGER     NOT NULL,     csreq          BLOB,     policy_id      INTEGER,     indirect_object_identifier_type    INTEGER,     indirect_object_identifier         TEXT NOT NULL DEFAULT 'UNUSED',     indirect_object_code_identity      BLOB,     flags          INTEGER,     last_modified  INTEGER     NOT NULL DEFAULT (CAST(strftime('%s','now') AS INTEGER)),     PRIMARY KEY (service, client, client_type, indirect_object_identifier),    FOREIGN KEY (policy_id) REFERENCES policies(id) ON DELETE CASCADE ON UPDATE CASCADE);
INSERT INTO access VALUES('kTCCServiceSystemPolicyAllFiles','/usr/sbin/smbd',1,2,4,1,X'fade0c000000002c0000000100000006000000020000000e636f6d2e6170706c652e736d6264000000000003',NULL,0,'UNUSED',NULL,0,1634341495);
VALUES('kTCCServiceSystemPolicyAllFiles','/Library/PrivilegedHelperTools/com.oracle.JavaInstallHelper',1,0,5,1,X'fade0c00000000a80000000100000006000000020000001c636f6d2e6f7261636c652e4a617661496e7374616c6c48656c706572000000060000000f000000060000000e000000010000000a2a864886f76364060206000000000000000000060000000e000000000000000a2a864886f7636406010d0000000000000000000b000000000000000a7375626a6563742e4f550000000000010000000a564235453254563936330000',NULL,NULL,'UNUSED',NULL,0,1636739681);
INSERT INTO access VALUES('kTCCServiceSystemPolicyAllFiles','Oracle.MacJREInstaller',0,0,5,1,X'fade0c00000000a4000000010000000600000002000000164f7261636c652e4d61634a5245496e7374616c6c65720000000000060000000f000000060000000e000000010000000a2a864886f76364060206000000000000000000060000000e000000000000000a2a864886f7636406010d0000000000000000000b000000000000000a7375626a6563742e4f550000000000010000000a564235453254563936330000',NULL,NULL,'UNUSED',NULL,0,1636739686);
INSERT INTO access VALUES('kTCCServiceDeveloperTool','com.apple.Terminal',0,0,4,1,NULL,NULL,0,'UNUSED',NULL,0,1669677542);
INSERT INTO access VALUES('kTCCServiceAccessibility','/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/AE.framework/Versions/A/Support/AEServer',1,0,4,1,X'fade0c000000003000000001000000060000000200000012636f6d2e6170706c652e4145536572766572000000000003',NULL,0,'UNUSED',NULL,0,1683241696);
INSERT INTO access VALUES('kTCCServiceSystemPolicyAllFiles','/usr/libexec/sshd-keygen-wrapper',1,2,4,1,X'fade0c000000003c0000000100000006000000020000001d636f6d2e6170706c652e737368642d6b657967656e2d7772617070657200000000000003',NULL,0,'UNUSED',NULL,0,1683310049);
INSERT INTO access VALUES('kTCCServicePostEvent','com.apple.screensharing.agent',0,2,4,1,NULL,NULL,0,'UNUSED',NULL,0,1683310059);
INSERT INTO access VALUES('kTCCServiceScreenCapture','com.apple.screensharing.agent',0,2,4,1,NULL,NULL,0,'UNUSED',NULL,0,1683310059);
INSERT INTO access VALUES('kTCCServiceSystemPolicyAllFiles','com.apple.Terminal',0,0,5,1,X'fade0c000000003000000001000000060000000200000012636f6d2e6170706c652e5465726d696e616c000000000003',NULL,NULL,'UNUSED',NULL,0,1684276653);
COMMIT;
sqlite> .exit
jbo@McJbo ~ %
```

Your output might be a bit different based on the system-wide `TCC.db` state, of course. Let's talk about the interesting bits:
- `service` is the type of access request; it'll start with `kTCCService`. There are many predefined types, and with each OS build it seems Apple adds more fine-grained TCC access request types.
- `client` is the client; it can be either the reverse-DNS macOS bundle name or a path.
- `client_type` is the type of client (`0` if for a bundle, `1` is for a path).
- `auth_value` is whether the request should be approved (`2`) or denied (`0`). There are also other values: `1` for unknown and `3` for limited.
- `auth_reason` is an enum specifying the reason behind the decision; will commonly be "User Set".
- `csreq` is a cryptographic blob that identifies the `client` - otherwise, anyone would be able to name their App with a specific reverse-DNS name and "inherit" the TCC entry.

There are other fields, but those are the ones that are relevant for this blogpost.

The per-user `TCC.db` file has the same schema but will commonly have different service types.  
Some service types are saved per-user (e.g. camera access) and some are global and will therefore persist in the system-wide `TCC.db` (e.g. Full Disk Access).  
Here are some common types:
- `kTCCServiceLiverpool`: Location services access, saved in the user-specific TCC database.
- `kTCCServiceUbiquity`: iCloud access, saved in the user-specific TCC database.
- `kTCCServiceSystemPolicyDesktopFolder`:	Desktop folder access, saved in the user-specific TCC database.
- `kTCCServiceCalendar`: Calendar access, saved in the user-specific TCC database.
- `kTCCServiceReminders`: Access to reminders, saved in the user-specific TCC database.
- `kTCCServiceMicrophone`: Microphone access, saved in the user-specific TCC database.
- `kTCCServiceCamera`: Camera access, saved in the user-specific TCC database.
- `kTCCServiceSystemPolicyAllFiles`: Full disk access capabilities, saved in the system-wide TCC database.
- `kTCCServiceScreenCapture`: Screen capture capabilities, saved in the system-wide TCC database.

## TCC as an attacker
Historically, there were many TCC bypasses; some folks ([@_r3ggi](https://twitter.com/_r3ggi), [@theevilbit](https://twitter.com/theevilbit)) find issues left and right and even offer some courses on the subject; from my experience, TCC bypasses range in severity significantly, so not all bypasses are created equal.  
As an attacker, the first thing I'd do is to see if I have `Full Disk Access` - having that means you could even edit the per-user `TCC.db` and, if you have root access - edit the system-wide database too! Here's a silly way to check if you have Full Disk Access - run the same `file` command from earlier and see if you get an error - if you don't then you probably have Full Disk Access!  
When it comes to real bypasses, I've seen several ideas:
1. Fooling `tccd` somehow; for example, in a vulnerability that [I reported](https://www.microsoft.com/en-us/security/blog/2022/01/10/new-macos-vulnerability-powerdir-could-lead-to-unauthorized-user-data-access/) I made the per-user `tccd` consume a different database that I plant ahead-of-time, essentially 
2. Running code in some other approved context; for example, [injecting into Zoom](https://www.csoonline.com/article/3535789/weakness-in-zoom-for-macos-allows-local-attackers-to-hijack-camera-and-microphone.html) immidiately gets you Camera and Microphone.
3. Running code in a pre-approved context; this is similar to the previous bulletpoint but injects into an Entitled process; some Entitlements can bypass TCC checks by-design (for instance, `tccd` obviously has `Full Disk Access` - think why it's esential and what the consequences of injecting into it would be!). I've talked about Entitlements [in the past](https://github.com/yo-yo-yo-jbo/macos_sip/) so be sure to get familiarized with the concept.
4. Other unique ideas; for example, [using TimeMachine backups](https://theevilbit.github.io/posts/cve_2020_9771/) to read the `TCC.db` file (and other files).

Note that TCC bypasses have been abused by malware in the past (e.g. [here](https://www.jamf.com/blog/zero-day-tcc-bypass-discovered-in-xcsset-malware/)), so if you find one make sure to disclosre responsibly!

## Summary
This was an introduction blogpost to TCC, which is another macOS security mechanism.  
While TCC can stop some attackers, it was proven in the past that it can be bypassed - I like to think of it as "defense-in-depth" that does not stand on its own.  
Nevertheless, Apple takes TCC bypasses very seriously, and will fix those bypasses for sure if you responsibly disclose. You might even get a bounty!

Stay tuned,

Jonathan Bar Or
