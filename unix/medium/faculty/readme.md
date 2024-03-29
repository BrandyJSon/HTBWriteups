### Faculty writeup (Created by gbyolo)

An nmap scan reveals ssh and http open. The http redirects to facutly.htb. We add an entry for it into our /etc/hosts and we see a page where we are asked to enter a faculty id number. We have no idea what the format for faculty ids are so I ran a feroxbuster before trying anything on the faculty login page. We find an endpoint that redirects to a admin login page.

![](./imgs/foothold/0.png)

The login page is vulnerable to sqli in the username field but finding it requires paying attention to teh raw responses of the server.

![](./imgs/foothold/0.1.png)

Testing for sqli using a single quote shows us the same error as if we enter wrong credentials.

![](./imgs/foothold/0.2.png)

However inspecting the traffic reveals it is likely sqli is possible.

![](./imgs/foothold/0.3.png)

We can test if both parameters are injectable by seeing if we get an error if we inject each parameter seperately. 

![](./imgs/foothold/0.4.png)

We can see password is not injectable most likely the password input is being hashed before being passed to the sql query. 

![](./imgs/foothold/0.5.png)

THe admin parameter is clearly injectable however. Testing a simple login bypass sqli with the username admin works. If admin did not exist though we could change the format of our injection using an or statment and a boolean statment that is always true. 

![](./imgs/foothold/1.png)

After we login we are redirected to an administrator panel for a school's scheduling. There are a number of different endpoints with different functionality. The most interesting being the faculty list and subject list endpoints which let us download pdfs.

![](./imgs/foothold/2.png)

Interectepting the request it appears we are able to control the pdf generated. As we send a request with a pdf postform field that contains double url encoded and base64 encoded pdf data.

![](./imgs/foothold/3.png)

After sending this request we are redirected to the generated pdf that exists in the mpdf directory. Mpdf is php library that is used to generate dynamic pdfs. Doing some research mpdf has a cve that allows for arbitrary file read in versions less then 7.0.0. I wasn't able to find any good ways to figure out the version of mpdf running so I figured I may as well try the exploit. 

![](./imgs/foothold/4.png)

The exploit allows for file read by adding a annotation file to the pdf with a local file on the system. We can test by trying to use the exploit to read /etc/passwd off the server.

![](./imgs/foothold/6.0.png)

Clearly the server is vulnerable. However as we will be probably need to read a lot of files and it is quite annoying to exploit this vulnerablilty manually like this lets quickly make a bash script to automate the process of reading the file.

```
#!/bin/bash

#file to read is $1

raw="<annotation file=\"$1\" content=\"$1\" icon=\"Graph\" title=\"Attached File: $1\" pos-x=\"195\" />"

encode=$(urlencode $raw | base64 | tr -d " \t\n\r")

filename=$(curl http://faculty.htb/admin/download.php -X POST -b "PHPSESSID=qcs9hac7urer83l9364bsvl8hm" -d "pdf=$encode" 2>/dev/null | tr -d " \t\n\r")

$(wget -q http://faculty.htb/mpdf/tmp/$filename -O /tmp/$filename 2>/dev/null)

if test -f "/tmp/$filename"; then
        file=$(./pdf-parser.py -f /tmp/$filename -o 6 | tail -n 2 | head -n 1)
        if [[ -z $file ]]; then
                file="b''"
        fi
        python3 -c "print($file.decode('utf-8'));"
        rm /tmp/$filename
else
        echo "File doesn't exist or caused an error"
fi 
```
This script takes the file to read as a command line argument then sends a request to the server to generate the pdf. It then downloads the file and uses a pdf-parser python script made by ![DidierStevens](https://github.com/DidierStevens) to extract the attached file printing it's contents. This link to the script is ![here](https://raw.githubusercontent.com/DidierStevens/DidierStevensSuite/master/pdf-parser.py). If the file cannot be read or does not exist it will instead print "File doesn't exist or caused an error". 

![](./imgs/foothold/7.png)

Reading the /etc/passwd file reveals two insteresting users: developer and gbyolo. Reading their ssh private keys unfrotunately is not possible.

![](./imgs/foothold/8.png)

Lets leak the source code for the webpage and start looking for another vulnerbaility so we can try to get rce on the machine. Most php websites have  adb_conn.php file that contains a password so we should also check for password reuse on the two users we found.

![](./imgs/foothold/9.png)

Checking the credentials on the users reveals that gbyolo has the same password as the db account letting us get commandline access to the box.

![](./imgs/foothold/10.png)

## User

As we cannot read the user flag as gbyolo so we need to look for priv esc. We have a password so lets see if we have an sudo permissons. 

![](./imgs/user/0.png)

We are able to sudo as the developer user and run meta-git, which is a node library that allows for anagement of meta repositories using git commands. Luckily looking for cve's on the library reveals a command injection vulnerability reported on hackerone by ![mik317](https://hackerone.com/mik317).

![](./imgs/user/1.png)

Looking at his report the file that the vulnerability exists in is metaGitUpdate.js. Lets read the metaGitUpdate.js file on the box and see if it has the same vulnerable code.

![](./imgs/user/1.1.png)

This snippet is clearly vulnerbale to command inject there being no sanitization of user inputs. Lets see if we can get the developer user's ssh private key using this technique.

![](./imgs/user/2.png)

Success!
With this we can reconnect through ssh as the developer user and get the first flag!

![](./imgs/user/3.png)

## Root

We unfortunately don't have as easy a time ahead for us getting root. There is no sudo permissons but we are a part of a group called debug. Using the find command reveals the only file with debug group permissons is GDB. (Always remeber to check for ACLs if find doesn't reveal any files when you are in a non standard group). Running a getCap reveals that GDB has the ![SYS_PTRACE](https://man7.org/linux/man-pages/man7/capabilities.7.html) capability set. 

![](./imgs/root/1.png)

Hacktricks has a very good page on exploiting linux capabilities. Looking under SYS_PTRACE gives us a python2.7 privilege escalation script that injects shellcode into the live execution of a process. I experimented for a while with trying to inject shellcode using GDB directly but I couldn't get the process to keep permissons. Perhaps this is becasue I don't have a lot of experience with GDB (go radare2!!) but whatever method I would try the process would always execute the shellcode as developer. Luckily for us the GDB on the box was compiled with python. Unfortuanetly it's python3 so we will need to update the script.

![](./imgs/root/2.png)

Just to clarify to anyone who like me who doesn't know the ins and outs of GDB. When GDB is compiled with python it allows for access to python terminal in the tool. However this does not spawn a new process, running inside GDB we will be able to run python command and our python script with the SYS_PTRACE capability. 

![](./imgs/root/3.png)

Now we have the the idea together. We need to fix the script so the python3 interpreter can run the code. Using a pip pacakage I found called ![2to3](https://docs.python.org/3/library/2to3.html) and making some slight manual work it wasn't too hard to fix the code. Additionally I changed the payload to ![this shellcode](https://www.exploit-db.com/shellcodes/45185) to avoid the repeated disappoint of popping a shell to only realize you are still the developer user from my GDB atempts.

![python3 verison of the script](./imgs/root/pyRoot2)

Following a blog post's recomendation on exploiting this capabilty our target process is going to be the nginx master process. We will hardcode the pid into the file because we are running the file in the python shell.

![](./imgs/root/4.png)

Now we need to open GDB and the priveleged python shell. We can run the file using the following syntax.

![](./imgs/root/injectWorks.png)

Checking /etc/passwd reveals the new user toor with a hash in /etc/passwd.

![](./imgs/root/6.png)

Switching users to toor we can finally root the box.

![](./imgs/root/8.png)

## Wrap up
This box was pretty fun though maybe a little easy for a medium box. The most interesting part was definetly root. There also was a couple rabbit holes. Most notiably was a further sqli on the get schedule endpoint on the admin page which didn't yield anyhting. The main thing the box lacked was enumeration. It felt very linear but besides that I would recomend at least going developer to root as playing around with gdb and injecting shellcode is quite fun.
