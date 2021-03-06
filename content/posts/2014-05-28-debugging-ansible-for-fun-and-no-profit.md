+++
title = "Debugging Ansible for fun and no profit"
date = 2014-04-28T15:19:00Z
+++
A colleague reported some strange behaviour regarding Ansible, in particular with `pgrep` and `pkill` in the shell module.
<!--more-->
I created a simplish test-case (I could have made it simpler but wanted to make it safish for other people to use)

```yaml
- hosts: localhost
  connection: local

  tasks:
  - name: generate random process string
    action: shell openssl rand 15 | base64
    register: process_string

  - name: warn user
    pause: prompt="About to kill processes matching {{process_string.stdout}} with signal SIGIO. Hit Return to continue or Ctrl-C then a to abort"

  - name: show bad effects of pkill -f
    action: shell pkill -IO -f {{process_string.stdout}} || true
```

Running this with `ansible-playbook pkilldemo.yml -v` shows the problem:

```yaml
PLAY [localhost] **************************************************************

GATHERING FACTS ***************************************************************
ok: [localhost]

TASK: [generate random process string] ****************************************
changed: [localhost] => {"changed": true, "cmd": "openssl rand 15 | base64 ", "delta": "0:00:00.011483", "end": "2014-05-28 15:46:36.753195", "rc": 0, "start": "2014-05-28 15:46:36.741712", "stderr": "", "stdout": "dkfCGgWBFr2ntK9uhSaw", "warnings": []}

TASK: [warn user] *************************************************************
[localhost]
About to kill processes matching dkfCGgWBFr2ntK9uhSaw with signal SIGIO. Hit Return to continue or Ctrl-C then a to abort:

ok: [localhost] => {"changed": false, "delta": 9, "rc": 0, "start": "2014-05-28 15:46:36.770642", "stderr": "", "stdout": "Paused for 0.16 minutes", "stop": "2014-05-28 15:46:46.223047", "user_input": ""}

TASK: [show bad effects of pkill] *********************************************
failed: [localhost] => {"changed": true, "cmd": "pkill -IO -f dkfCGgWBFr2ntK9uhSaw || true ", "delta": "0:00:00.010195", "end": "2014-05-28 15:46:46.379405", "rc": -29, "start": "2014-05-28 15:46:46.369210", "warnings": []}

FATAL: all hosts have already failed -- aborting

PLAY RECAP ********************************************************************
           to retry, use: --limit @/home/will/pkilldemo.retry

           localhost                  : ok=3    changed=1    unreachable=0    failed=1 
```

So running `pkill -IO -f pattern_to_search_for || true` returns `-29`
(where `SIGIO` is signal `29` on my machine).

Adding the following to the above playbook allows us to see different
success and failure scenarios:

```yaml
    ignore_errors: True

  - name: works ok without -f
    action: shell pkill -IO {{process_string.stdout}} || true

  - name: works ok with pgrep -f
    action: shell pgrep -f {{process_string.stdout}} || true

  - name: this is supposed to fail as we don't handle the error with true
    action: shell pgrep -f {{process_string.stdout}}
    ignore_errors: True

  - name: what we should actually run
    action: raw pkill -f {{process_string.stdout}}
    ignore_errors: True
```

After running the resulting playbook (that playbook and output are
[available as a gist](https://gist.github.com/willthames/ee40bd6d9b5eebb9b8eb)),
the known facts are these:

* Running `pkill -IO -f pattern_to_search_for || true` on the host itself
outside of ansible exits with status 0
* Running `pkill -IO pattern_to_search_for || true` through Ansible returns 0
* Running `pgrep -f pattern_to_search_for || true` through Ansible returns 0
* Running `pkill -IO -f pattern_to_search_for` through Ansible
exit status 1, a known exit code according to `man pkill`

## Setting up debugging in Ansible
I tend to use pdb for debugging with the python script that actually gets generated by Ansible. The creator of
Ansible, Michael Dehaan ([@laserllama](https://twitter.com/laserllama)) suggests
[epdb for Ansible debugging](http://michaeldehaan.net/post/35403909347/tips-on-using-debuggers-with-ansible).

As Michael suggests in that blogpost, it's very important to set forks to 1. You can do this
temporarily using `--forks=1` on the command line but I tend to do so little stuff that needs parallelism
that I have `forks = 1` in my ansible config file!

To obtain the module python script, you can set the `ANSIBLE_KEEP_REMOTE_FILES` environment variable, either
using `export ANSIBLE_KEEP_REMOTE_FILES=1` or

```yaml
[will@fedora pkilldemo]$ ANSIBLE_KEEP_REMOTE_FILES=1 ansible-playbook pkilldemo.yml -vvv

PLAY [localhost] **************************************************************

GATHERING FACTS ***************************************************************
<snip>

TASK: [generate random process string] ****************************************
<snip>

TASK: [warn user] *************************************************************
<snip>

TASK: [show bad effects of pkill -f] ******************************************
<localhost> REMOTE_MODULE command pkill -IO -f Ki5ViZNY4EIRaf2JDqvQ || true #USE_SHELL
<localhost> EXEC ['/bin/sh', '-c', 'mkdir -p $HOME/.ansible/tmp/ansible-tmp-1401256918.15-50781745619374 && chmod a+rx $HOME/.ansible/tmp/ansible-tmp-1401256918.15-50781745619374 && echo $HOME/.ansible/tmp/ansible-tmp-1401256918.15-50781745619374']
<localhost> PUT /tmp/tmp39O8iv TO /home/will/.ansible/tmp/ansible-tmp-1401256918.15-50781745619374/command
<localhost> EXEC ['/bin/sh', '-c', u'LC_CTYPE=en_US.UTF-8 LANG=en_US.UTF-8 /usr/bin/python /home/will/.ansible/tmp/ansible-tmp-1401256918.15-50781745619374/command']
failed: [localhost] => {"changed": true, "cmd": "pkill -IO -f Ki5ViZNY4EIRaf2JDqvQ || true ", "delta": "0:00:00.017252", "end": "2014-05-28 16:01:58.302777", "rc": -29, "start": "2014-05-28 16:01:58.285525", "warnings": []}
```

So I can then run `python  /home/will/.ansible/tmp/ansible-tmp-1401256918.15-50781745619374/command` to repeat the task
```sh
[will@fedora pkilldemo]$ python  /home/will/.ansible/tmp/ansible-tmp-1401256918.15-50781745619374/command
{"changed": true, "end": "2014-05-28 16:03:49.657242", "stdout": "", "cmd": "pkill -IO -f Ki5ViZNY4EIRaf2JDqvQ || true ", "start": "2014-05-28 16:03:49.646183", "delta": "0:00:00.011059", "stderr": "", "rc": -29, "warnings": []}
```

At this point it's probably worth a brief exposition of how the shell
module works in Ansible.
Really it just kicks off the `command` module with a `USE_SHELL=1`
module. So I'm going to be
looking at the [command module source](https://github.com/ansible/ansible/blob/bb3426327c2d612b0740e9c644ea45535a3f3a0f/library/commands/command)
from the latest commit at time of writing.

What Ansible does with that module is to replace line 162 with the contents of
the [module_utils/basic code](https://github.com/ansible/ansible/blob/bb3426327c2d612b0740e9c644ea45535a3f3a0f/lib/ansible/module_utils/basic.py)
and then swap a few template tokens (e.g. `"<<INCLUDE_ANSIBLE_MODULE_ARGS>>"` gets replaced with  `'pkill -IO -f Ki5ViZNY4EIRaf2JDqvQ || true #USE_SHELL'`).
Anyway, the end result is that the 213 line command module gets expanded to 1419 lines.

The line I'm interested is the same either way - it's line 140,
where the command gets executed. So I open up the script and add

```python
import pdb
pdb.set_trace()
```

just before line 140, and run it again.

The python docs have some good instructions on [how to use the debugger](https://docs.python.org/2/library/pdb.html#debuggr-commands).
After stepping through the debugger enough, I realise the eventual
result is that python is going to kick off a shell that looks like

```sh
sh -c 'pkill -IO -f Ki5ViZNY4EIRaf2JDqvQ || true'
```
And, sure enough, that matches itself, and kills itself before it
gets to the `true`:

```sh
[will@fedora ansible (devel)]$ sh -c 'pkill -IO -f bobbins || true'
I/O possible
[will@fedora ansible (devel)]$ echo $?
157
```

where 157 is 128 + 29.

<blockquote class="blockquote-reverse">
<p>When a command terminates on a fatal signal N, bash uses the value of 128+N as the exit status</p>
<footer><code>man bash</code></footer>
</blockquote>

This explains why `pgrep` does not fail, and why `pkill` without `-f`
does not fail.
It's not really an Ansible bug (one could argue that maybe process
management is a missing module), but it was instructive to use Ansible
debugging techniques to get to the cause.

And the actual solution to the problem was to use `ignore_errors` rather
than using `true` in the shell to hide the error.

```yaml
  name: kill process if running
  action: command pkill -f bobbins
  ignore_errors: True
```

