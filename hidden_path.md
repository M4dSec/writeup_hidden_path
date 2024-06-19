# Overall Challenge Information
CTF: `HTB Business CTF 2024: The Vault Of Hope`

Name: Hidden Path
Level: Easy
Category: Misc
Points: 300 points
Solves: 170

Challenge Description: Legends speak of the infamous Kamara-Heto, a black-hat hacker of old who rose to fame as they brought entire countries to their knees. Opinions are divided over whether the fabled figure truly existed, but the success of the team surely lies in the hope that they did, for the location of the lost vault is only known to be held on what remains of the NSA's data centres. You have extracted the source code of a system check-up endpoint - can you find a way in? And was Kamara-Heto ever there?

# Initial Analysis
We have a Docker container running on `83.136.253.105:47045` and a zip file `misc_hidden_path.zip` containing the source code for that Docker container.

In the source files, we see a Node.js application that seems to be running on port `47045`. We can confirm this by using `curl` on the port:
```
wkzshi@deadsec:~$ curl 83.136.253.105:47045
<!DOCTYPE html>

<head>
    <title>NSA Server System Checker</title>
    <script src="/js/index.js"></script>
    <link rel="stylesheet" href="/css/cyberpunk.css">
</head>

<body>
    <div class="center">
        <h1 class="cyber-h oxanium-font ac-purple">Check NSA Server Status</h1>
        <form name="commandForm" onsubmit="return submitCommand()">
            <input type="radio" id="free" class="cyber-radio ac-purple oxanium-font" name="command" value="free -m">
            <label for="free">free -m</label><br><br>
            <input type="radio" id="uptime" class="cyber-radio ac-purple oxanium-font" name="command" value="uptime">
            <label for="uptime">uptime</label><br><br>
            <input type="radio" id="iostat" class="cyber-radio ac-purple oxanium-font" name="command" value="iostat">
            <label for="iostat">iostat</label><br><br>
            <input type="radio" id="mpstat" class="cyber-radio ac-purple oxanium-font" name="command" value="mpstat">
            <label for="mpstat">mpstat</label><br><br>
            <input type="radio" id="netstat" class="cyber-radio ac-purple oxanium-font" name="command" value="netstat">
            <label for="netstat">netstat</label><br><br>
            <input type="radio" id="ps" class="cyber-radio ac-purple oxanium-font" name="command" value="ps aux">
            <label for="ps">ps aux</label><br><br>
            <button class="cyber-button bg-purple fg-white m-2 vt-bot oxanium-font" type="submit">Check System</button>
        </form>
        <br>
        <div class="code-block" id="response"></div>
    </div>
</body>
</html>
```

That's right, let's see the app!


We see a simple web application that appears to be a server status checking system.

![](../../../../../../assets/htb/hidden_path/main_page.png)

The radio buttons and 'Check System' button suggest that we can send commands to be processed by the server and view the output. Let's send some commands and investigate the responses:

![](../../../../../../assets/htb/hidden_path/command.png)

Each command provides some information, but one response caught my attention immediately:

![](../../../../../../assets/htb/hidden_path/command2.png)

In the response line `PID USER TIME COMMAND 1 root 0:00 node app.js 18 root 0:00 ps aux`, we see that all commands are executed by `root`, which is an intriguing find.

Next, I launched my Burp Suite for further investigation.

# Looking at the requests

The first thing I wanted to investigate was the actual request with the command being sent to the server. We see `POST` requests sent to the `/server-status` endpoint:

![](../../../../../../assets/htb/hidden_path/status.png)

I sent one of these requests in Burp Repeater for a detailed investigation, and it appeared quite minimalistic:

![](../../../../../../assets/htb/hidden_path/init_post.png)

There is only one parameter choice with the numeric value of our option. This gives us some insight into the application's internal structure. Let's examine the source code.

# Source code analysis

The app has a simple Node.js structure with `Dockerfile`:

![](../../../../../../assets/htb/hidden_path/tree.png)

There is a flag.txt provided, indicating the flag will be in the same directory on the server. After analyzing the source code, I checked the `app.js` file and saw something very interesting in the `POST` request handler function:

![](../../../../../../assets/htb/hidden_path/legendary.png)

Do you see it? There is a hidden character in the line `const {choice, } = req.body`:

![](../../../../../../assets/htb/hidden_path/upclose.png)

JavaScript has a feature called **assignment destructuring**, which allows developers to assign multiple variables in a single line of code. This function is used in the line `const {choice, } = req.body`, and this little symbol is a valid variable name. So, in theory, we could pass a different value in the `POST` request, and that value would be assigned to that strange `"ㅤ"` variable.

Moreover, this same symbol also appears in the `command` list, which indicates which commands we are allowed to run on the server:

![](../../../../../../assets/htb/hidden_path/commands.png)

Do you see where this is going? We can pass a `*some-command*` value to that character variable, and it will end up in the list of allowed commands. Let's exploit this hidden vulnerability.

# Exploitation of a Null Character

In my Burp Repeater, I tried several requests with different values, and after some trials, I got this response:

![](../../../../../../assets/htb/hidden_path/root.png)

We can pass another parameter with name of `ㅤ`, value of our new command, and execute it with `choice=6`. RCE is officially found! Now it's just a matter of exposing the flag, and we already know where it is thanks to the source code.
The simplest way to do this is by using `cat ./flag.txt`. However, we need to URL encode our command first. I used `Cyberchef` for this:

![](../../../../../../assets/htb/hidden_path/cyberchef.png)

Now, we have a mighty payload for the `POST` request: `choice=6&ㅤ=cat%20%2E%2Fflag%2Etxt`. Let's try it out!

![](../../../../../../assets/htb/hidden_path/flag.png)

There it is.

# Conclusion

The challenge presented a fun and educational experience, emphasizing the importance of understanding web application vulnerabilities and the power of detailed source code analysis. The hidden null character provided an unexpected twist, showcasing how even minor details can be exploited in many creative ways.
