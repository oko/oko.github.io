---
layout: post
title: "PHP-FPM Socket Passing"
---

As noted on the [PHP](https://wiki.php.net/rfc/socketactivation) and [Systemd](http://freedesktop.org/wiki/Software/systemd/DaemonSocketActivation/) wikis, PHP-FPM uses the `FPM_SOCKETS` variable to allow socket inheritance during reloads. While those examples (and all others I was able to find) use domain sockets, it's also possible to pass arbitrary TCP sockets with minimal fuss.

All that's required to launch PHP-FPM this way is an environment variable in the format `FPM_SOCKETS=<ipaddress>:<port>=<fileno>` and a matching `listen` directive in PHP-FPM's configuration file. The match between the environment variable and configuration file is required as PHP-FPM only uses inherited sockets that have corresponding `listen` directives. This can be taken care of with a little bit of `sed` or by generating a configuration file template on-demand.

As an example, let's pass a random port opened initially in Python to PHP-FPM:

```python
# Bind the initial socket (port 0 = auto-choose)
import socket
s = socket.socket()
s.bind(('127.0.0.1', 0))
s.listen(5)

# Get socket/file descriptor information
addr, port = s.getsockname()
fd = s.fileno()

# Update configuration
subprocess.call(["/usr/bin/sed", "-i",
                 "s/^listen[ ]*=.*/listen = %s:%d/" % (addr, port),
                 "/etc/php-fpm.conf"])

# Exec PHP-FPM in-place
os.execve("/usr/sbin/php-fpm", ["/usr/sbin/php-fpm"], {'FPM_SOCKETS': '%s:%d=%d' % (addr, port, fd)})
```
