---
title: "Tweak Python for Windows to use less of your C Drive"
date: 2020-04-15T16:15:17+01:00
draft: true
tags: ["development", "python"]
---

Python's Windows installer is very good nowadays, but there are a couple
of tweaks you can apply to make it easier for Python to use less of your
C Drive, which is especially useful if you have a small SSD.

What I end up doing is:

**Step 1. Install Python to D Drive.** For instance, a folder like
D:\\*ProgramsDir*\Python\Python38. I typically *don't* add Python to the PATH
because I have a [doskey alias](https://superuser.com/a/1134468/2576) that adds
the required directories to my PATH inside of a Command Prompt, but people who
use a lot of Python may find it useful to let the installer add Python (and
Scripts, etc) to the PATH.

**Step 2. Store per-user site-packages on D.** Set the user environment
variable `PYTHONUSERBASE` to point the per-user site-packages directory to, for
instance, `D:\Data\PyData`. See [PEP
370](https://www.python.org/dev/peps/pep-0370/) for more details.

**Step 3. Virtual Environments.** The other thing that chews up space on C: is
virtual environments.  To get virtual environments on Windows, I install
`virtualenvwrapper-win` (using pip). I then set the user environment variable
`WORKON_HOME` to `D:\Data\PyEnvs`. Virtualenvs are now created under this
directory.
