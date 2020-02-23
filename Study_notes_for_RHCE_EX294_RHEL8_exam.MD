# Study notes for EX294 / EX407 Ansible Automation exam (RHEL8)
_by Tomas Nevar (tomas@lisenet.com)
_adapted by Mircea Niscoveanu

## Exam objectives:

* Understand core components of Ansible
  - Inventories
  - Modules
  - Variables
  - Facts
  - Plays
  - Playbooks
  - Configuration files
* Install and configure an Ansible control node
  - Install required packages
  - Create a static host inventory file
  - Create a configuration file
* Configure Ansible managed nodes
  - Create and distribute SSH keys to managed nodes
  - Configure privilege escalation on managed nodes
  - Validate a working configuration using ad-hoc Ansible commands
* Create simple shell scripts that run ad hoc Ansible commands
* Use both static and dynamic inventories to define groups of hosts
* Utilize an existing dynamic inventory script
* Create Ansible plays and playbooks
  - Know how to work with commonly used Ansible modules
  - Use variables to retrieve the results of running commands
  - Use conditionals to control play execution
  - Configure error handling
  - Create playbooks to configure systems to a specified state
* Use Ansible modules for system administration tasks that work with:
  - Software packages and repositories
  - Services
  - Firewall rules
  - File systems
  - Storage devices
  - File content
  - Archiving
  - Scheduled tasks
  - Security
  - Users and groups
* Create and use templates to create customized configuration files
* Work with Ansible variables and facts
* Create and work with roles
* Download roles from an Ansible Galaxy and use them
* Manage parallelism
* Use Ansible Vault in playbooks to protect sensitive data
* Use provided documentation to look up specific information about Ansible modules and commands


## 0. RHEL 8 Useful commands:

  ```
  $ systemctl shutdown
  $ systemctl reboot
  ```
 
### YUM / DNF

* List modules: 

	`$ yum module list` - shows all versions for each module
* Enable another version: 

	`$ yum module enable postgresql:12`
	
* Reset module version to default [d]: 

	`yum module reset postgresql` 
	
* Other yum module arguments : 

	`$ yum module {update,provides,install,reset,disable,enable,info,remove,list} MODULE_NAME`
	
### Interrupt boot process and Change the root password:

* Edit the kernel boot parameters by pressing "`e`"
* Go to the end of "linux" line by pressing `CTR+e` removing `ro crash` and add `rd.break enforcing=o`
* Start the system by pressing `CTR+x`
* Remount the root of the system: 

  `$ mount -o remount,rw /sysroot`
  
* Swith to /sysroot: `$ chroot /sysroot`
* Reset the “root” password: `$ passwd`
* Enable SElinux relabeling: `$ touch /.autorelabel`
* Exit the SHELL:   ```$ exit```

### Disk management:

  ```$ blkid``` -> shows UID of filesystems

### Mount a Network File System
	
	$ yum install nfs-utils
	$ systemctl start rpcbind
	$ systemctl start nfs-server (on server)
	
* Show file system exports on the client:

`$ showmount -e SERVER_IP` (on client)
	
* Mount nfs:

`$ mount -t nfs SERVER_IP:/nfs-share /mounted-nfs-share` (on client)
		
/etc/fstab: `SERVER_IP:/nfs-share /mounted-nfs      nfs4    defaults 0 0` (on client)
	
/etc/exports: `/nfs-share CLIENT_IP(rw,sync, permissions etc)` (on server)
  
### Special permissions:

* `setuid`  (only to files ) – when running the executable, it runs with the owner permission and not with the actual user that runs the executable
* `setgid` (files and folders) – if set , the files/folders created under the parent will have the group of the of the parent folder
* `sticky bit` (folders) – if set , the files under folder can only be deleted by the owner

### Working with Virtual Data Optimizer (VDO)

* Install required package: `yum install vdo`

* Create a VDO volume:

	`$ vdo create --name=vdo_volume_1 --device=/dev/devName --vdoLogicalSize=vol_size`

* View information on vdo volumes:

`$ vdostats --human-readable`

### Configure time service client : chrony

* Install required package: `yum install chrony`
* Start "chronyd": `systemctl start chronyd && systemctl enable chronyd`
* Configure "chronyd": `vi /etc/chronyd`
* Restart "chornyd" to apply configuration: `systemctl restart chronyd`

### Users and Groups

`$ id USER_NAME`

`$ groups USER_NAME` -> show belonging groups

`$ usermod -aG NEW_GROUP_TO_ADD USER_NAME` - add the user to a new group

`$ usermod -g NEW_MAIN_GROUP USER_NAME` - sets the new main group

`$ usermod -L USER_NAME` - lock user

`$ usermod -U USER_NAME` - unlock user

* Change password to user:

`$ passwd USER_NAME` - to change password for user

`$ chage USER_NAME` - to set password expiracy policies

### SElinux 

`$ getenforce` - view selinux mode [0|1] or [Permissive|Enforcing]

`$ setenforce [0|1]` - set selinux mode

`$ setatus` - view SElinux status

* List boolenas: `$ getsebool -a`
* Turn boolean ON of OFF: `$ setsebool BOOLEAN_NAME on | off -P` - add "-P" for permanent
* List SElinux contexts: `$ semanage fcontext -l`
* View contexts on files of folders:

	```
	$ ls -Z
	$ ps -axZ
	```
	
* Change SElinux context (semanage or chcon or setfiles):

	`$ semanage fcontext -a -t context_type '/var/www/html(./*)?'` - changes SElinux context recurrent
	
* Restore default contexts: `$ restorecon -R /var/www/html`

* View SElinux policy violations:
	`$ seaalert -a /var/log/audit/audit.log`

#### Ansible SElinux useful modules

**selinux**
```
- name: Enable SELinux
  selinux:
    policy: targeted
    state: enforcing
```

**seboolean**
```
- name: Set httpd_can_network_connect flag on and keep it persistent across reboots
  seboolean:
    name: httpd_can_network_connect
    state: yes
    persistent: yes
```

**sefcontext**
```
- name: Allow apache to modify files in /srv/git_repos
  sefcontext:
    target: '/srv/git_repos(/.*)?'
    setype: httpd_git_rw_content_t
    state: present

- name: Apply new SELinux file context to filesystem
  command: restorecon -irv /srv/git_repos
```

## 1. Ansible Getting Started

### Installing on RedHat

	
	$ subscription-manager repos list | grep ansible
	$ subscription-manager repos --enable ansible-2.9-for-rhel-8-x86_64-rpms
	$ yum install ansible
	

When installed, the ansible package provides a base configuration file located at `/etc/ansible/ansible.cfg`.

Priority in which the configuration files are processed:
1) `$ANSIBLE_CONFIG` (an environment variable)
2) `./ansible.cfg` (in the current directory)
3) `~/.ansible.cfg` (the user's home directory)
4) `/etc/ansible/ansible.cfg`

To find out what config file is in use, run the following:
```
$ ansible --version
ansible 2.7.9
  config file = /home/ansible/ansible.cfg
```
To get started, copy configuration files to the home directory:
```
$ cp /etc/ansible/ansible.cfg ~/ansible.cfg
$ cp /etc/ansible/hosts ~/inventory
```
Note the maximum number of simultaneous connections that Ansible makes is controlled by the forks parameter in `ansible.cfg`:
```
forks 5
```
Example inventory configuration:
```
$ cat ~/inventory
[myself]
127.0.0.1 ansible_connection=local
[web]
ansible[2:3].hl.local
[db]
ansible[4:5].hl.local ansible_user=dbadmin
[lab:children]
web
db
```
There is a special group named "all" that matches all managed hosts in the inventory.
```
- hosts: all
```
There is also a special group named "ungrouped" which matches all managed hosts in the inventory that are not members of any group.
```
- hosts: ungrouped
```
Quote host patterns used on the CLI to protect them from unwanted shell expansion:
```
- hosts: '*.hl.local'
- hosts: '10.11.1.*'
```
Multiple entries in the inventory can be referenced using lists:
```
- hosts: web,ansible3.hl.local
```
If you have static and dynamic inventory files in the same directory, then they are merged and treated as one inventory!

Show inventory for all:
```
$ ansible "*" -i ~/inventory --list-hosts
```

Privilege escalation configuration in `ansible.cfg`:
```
[privilege_escalation]
become = false
become_method = sudo
become_user = root
become_ask_pass = false
```

How to set the default user to use for playbooks in `ansible.cfg`:
```
remote_user = ansible
```
How to set the log file in `ansible.cfg`:
```
log_path = /home/ansible/ansible.log
```

If no module is defined, Ansible uses the internally predefined "command" module. Note that this is not a shell!
Ad hoc command that generates one line input for each operation:
```
$ ansible all -m command -a /usr/bin/hostname -o
```
Use the copy module to change content of a file:
```
$ ansible all -m copy \
  -a 'content="Host managed by Ansible\n" dest=/etc/motd' \
  -u ansible --become
```

Note: if possible, try to avoid the command, shell and raw modules in playbooks! It's easy to write non-idempotent playbooks this way.

Ansible module documentation and playbook snippets:
```
$ ansible-doc -l
$ ansible-doc -s <module_name>
$ ansible-doc <module_name>
```

Ansible executes plays and tasks in the order they are presented! How to check for YAML syntax errors:
```
$ ansible-playbook --syntax-check <filename>.yaml
```
Which host(s) the playbook applies to?
```
$ ansible-playbook <playbook>.yaml --list-hosts
```
Whats tasks will be performed?
```
$ ansible-playbook <playbook>.yaml --list-tasks
```
Run one task at a time:
```
$ ansible-playbook <playbook>.yaml --step
```
Playbook dry-run:
```
$ ansible-playbook -C <playbook>.yaml
```
Playbook step-by-step execution:
```
$ ansible-playbook --step <playbook>.yaml
```

The beginning of each play begins with a single dash followed by a space.

Playbook attributes:
* name - can be used to give a descriptive label to a play.
* hosts - must be defined in every play.
* remote_user - can be used to define the user that runs the tasks.
* become - can be used to enable privilege escalation.
* become_method - can be used to define the privilege escalation method.
* become_user - can define the user account to be used for privilege escalation.
* tasks - is defined as a list of dictionaries.
* blocks - can be used to group related tasks together.

For each play in a playbook, you get to choose which machines in your infrastructure to target and what remote user to complete the steps as. You can use keyword `become` on a particular task instead of the play:
```
---
- hosts: webservers
  remote_user: ansible
  serial: 2
  tasks:
    - service:
        name: httpd
        state: started
      become: yes
      become_method: sudo
```
Use the serial keyword to run the hosts through the play in batches. Host variables take precedence over group variables, but variables defined by a playbook take precedence over both.

## 2. Including and Importing Files

When you include content, then Ansible processes included content during the run of the playbook, as content is required.
When you import content, Ansible processes imported content when the playbook is initially read, before the run starts.

Note that `include` was replaced in Ansible 2.4 with new directives such as `include_tasks`, `include_vars`, `include_role`.

```
import_playbook:
import_tasks:
include_tasks:
```
Examples:
```
tasks:
  - name: Import task file and use variables
    import_tasks: task.yml
    vars:
      package: vsftpd
      service: vsft


  - name: Install the {{ package }} package
    yum:
      name: "{{ package }}"
      state: latest
  - name: Start the {{ service }} service
    service:
      name: "{{ service }}"
      enabled: true
      state: started
```

#### Dynamic vs. Static

Ansible has two modes of operation for reusable content: dynamic and static.

In Ansible 2.0, the concept of dynamic includes was introduced. Due to some limitations with making all includes dynamic in this way, the ability to force includes to be static was introduced in Ansible 2.1. Because the include task became overloaded to encompass both static and dynamic syntaxes, and because the default behavior of an include could change based on other options set on the Task, Ansible 2.4 introduces the concept of `include` vs. `import`.

If you use any `include*` Task (`include_tasks`, `include_role`, `include_vars`), it will be **dynamic**. 
If you use any `import*` Task (`import_playbook`, `import_tasks`, `import_role`), it will be **static**.

	import_playbook – Import a playbook
	import_role – Import a role into a play
	import_tasks – Import a task list
	include – Include a play or task list
	include_role – Load and execute a role
	include_tasks – Dynamically include a task list
	include_vars – Load variables from files, dynamically within a task


The bare `include` task (which was used for both Task files and Playbook-level includes) is still available, however it is now considered **deprecated**.

#### Differences Between Dynamic and Static

The two modes of operation are pretty simple:

   Dynamic includes are processed during runtime at the point in which that task is encountered.
   Ansible pre-processes all static imports during Playbook parsing time.

When it comes to Ansible task options like tags and conditional statements (when:):


   For dynamic includes, the task options will only apply to the dynamic task as it is evaluated, and will not be copied to child tasks.
   For static imports, the parent task options will be copied to all child tasks contained within the import.


**Note:** 
Roles are a somewhat special case. Prior to Ansible 2.3, roles were always statically included via the special roles: option for a given play and were always executed first before any other play tasks (unless pre_tasks were used). Roles can still be used this way, however, Ansible 2.3 introduced the `include_role` option to allow roles to be executed inline with other tasks.

Using `include*` vs. `import*` has some advantages as well as some tradeoffs which users should consider when choosing to use each:

The primary advantage of using `include*` statements is looping. When a loop is used with an include, the included tasks or role will be executed once for each item in the loop.

Using `include*` does have some limitations when compared to `import*` statements:

	
   Tags which only exist inside a dynamic include will not show up in `--list-tags` output.
   Tasks which only exist inside a dynamic include will not show up in `--list-tasks` output.
   You cannot use notify to trigger a handler name which comes from inside a dynamic include (see note below).
   You cannot use --start-at-task to begin execution at a task inside a dynamic include.
	

Using `import*` can also have some limitations when compared to dynamic includes:


   As noted above, loops cannot be used with imports at all.
   When using variables for the target file or role name, variables from inventory sources (host/group vars, etc.) cannot be used.
   Handlers using `import*` will not be triggered when notified by their name, as importing overwrites the handler’s named task with the imported task list.


**Note:** 
Regarding the use of notify for dynamic tasks: it is still possible to trigger the dynamic include itself, which would result in all tasks within the include being run.

## 3. Ansible and Vim

When writing playbooks in vim editor, modify its action in response to Tab key entries. Add the following line to `$HOME/.vimrc`. It will perform a two space indentation when the Tab key is pressed.
```
autocmd FileType yaml setlocal ai ts=2 sw=2 et
```
Settings explained:
```
ai   - autoindent (turns it on)
sw=2 - shiftwidth (indenting is 2 spaces)
et   - expandtab (do not use actual tab character)
ts=2 - tabstop (tabs are at proper location)
```
Two other helpful but optional vim settings:
```
nu (show line number)
cursorline (line to show current cursor position)
```
## 4. Ansible Vault
```
$ ansible-vault [create|decrypt|edit|encrypt|rekey|view] vaultfile.yml
```
By default, Ansible uses functions from the `python-crypto` package to encrypt and decrypt vault files. To speed up decryption at startup, you install the `python-cryptography` package.
```
$ ansible-vault create vaultfile.yml
$ ansible-vault view vaultfile.yml
```
Check syntax of a playbook that uses the vault file:
```
$ ansible-playbook --syntax-check --ask-vault-pass playbook.yml
```
Create a password file to use for the playbook execution:
```
$ echo "ansible" > key
$ ansible-playbook --syntax-check --vault-password-file=key playbook.yml
```

## 5. Ansible Facts
The same thing as facts in Puppet. How to print facts for all hosts?
```
- name: Ansible fact dump
  hosts: all
  tasks:
    - name: Print all Ansible facts
      debug:
        var: ansible_facts
```
```
- name: Ansible package fact dump
  hosts: all
  tasks:
    - name: Gather package facts
      package_facts:
        manager: auto
```
Before Ansible 2.5, facts were injected as individual variables prefixed with the string `ansible_` instead of being part of the `ansible_facts` variable.

At the time of writing this, Ansible recognises both the new fact naming system (using `ansible_facts`) and the old pre 2.5 naming system where facts are injected as separate variables.

You can use an ad-hoc command to run the setup module to print the value of all facts:
```
$ ansible localhost -m setup
```
Filter results:
```
$ ansible localhost -m setup -a 'filter=*distribution*'
```
Some commonly used facts (from my experience with Puppet). In no particular order:
```
ansible_facts['hostname']
ansible_facts['fqdn']
ansible_facts['default_ipv4']['address']
ansible_facts['distribution']
ansible_facts['distribution_major_version']
ansible_facts['domain']
ansible_facts['memtotal_mb']
ansible_facts['processor_count']
```

To disable fact gathering for a play, you can set the `gather_facts` keyword to "no":
```
gather_facts: no
```
Similar to Puppet, Ansible supports custom facts. Custom facts can be defined in a static file, formatted as an INI file or using JSON and placed in `/etc/ansible/facts.d` and the file name must end in `.fact`. They can also be executable scripts which generate JSON output, just like a dynamic inventory script. Note that custom fact files cannot be in YAML format!

## 6. Ansible Loops, Conditionals and Lookups

Ansible supports iterating a task over a set of items using the `loop` keyword. A simple loop iterates a task over a list of items.

Since Ansible 2.5, the recommended way to write loops is to use the `loop` keyword. The old syntax used loops prefixed with `with_`.
```
- name: Ensure that packages are present
  package:
    name: "{{ item }}"
    state: present
  loop:
    - firewalld
    - httpd
    - vsftpd
```
```
- name: Ensure that web server ports are open
  firewalld:
    service: "{{ item }}"
    immediate: true
    permanent: true
    state: enabled
  loop:
    - http
    - https
```
Example conditionals:
```
Equal				|	==
Less than			|	<
Greater than 			|	>
Variable exists 		|	is defined
Variable does not exist		|	is not defined
Boolean is true			|	values of 1, True, or yes evaluate to true
```
Logical AND and OR operations are supported:
```
when: ansible_domain == "hl.local" and ansible_distribution == "RedHat"
when: ansible_processor_cores == "1" or ansible_processor_cores == "2"
```
Loops and conditionals can be combined:
```
- name: Ensure that packages are present on RedHat
  package:
    name: "{{ item }}"
    state: present
  loop:
    - firewalld
    - httpd
  when: ansible_distribution == "RedHat"
```
Lookup plugins allow access to outside data sources. Lookups occur on the local computer, not on the remote computer. One way of using lookups is to populate variables:
```
vars:
  motd_value: "{{ lookup('file', '/etc/motd') }}"
tasks:
  - debug:
      msg: "motd value is {{ motd_value }}"
```

## 7. Ansible Handlers
Handlers always run in the order specified by the handlers section of the play. A handler called by a task in the tasks part of the playbook will not run until all of the tasks under tasks have been processed.
```
notify: restart httpd service

handlers:
  - name: restart httpd service
    service:
      name: httpd
      state: restarted
```
If a task fails and the play aborts on that host, any handlers that had been notified by earlier tasks in the play will not run. Use the following to force execution of the handler:
```
force_handlers: yes
```
Handlers are notified when a task reports a "changed" result. Handlers are not notified when it reports an "ok" or "failed" result.

Ignore errors:
```
ignore_errors: yes
```

You can change this behavior with the `--force-handlers` command-line option, or by including `force_handlers: True` in a play, or `force_handlers = True` in ansible.cfg. When handlers are forced, they will run when notified even if a task fails on that host. (Note that certain errors could still prevent the handler from running, such as a host becoming unreachable.)

## 8. Error Handling

#### Controlling What Defines Failure

Ansible lets you define what "failure" means in each task using the `failed_when` conditional. As with all conditionals in Ansible, lists of multiple `failed_when` conditions are joined with an implicit and, meaning the task only fails when all conditions are met. If you want to trigger a failure when any of the conditions is met, you must define the conditions in a string with an explicit or operator.

You may check for failure by searching for a word or phrase in the output of a command:
```
- name: Fail task when the command error output prints FAILED
  command: /usr/bin/example-command -x -y -z
  register: command_result
  failed_when: "'FAILED' in command_result.stderr"
```
or based on the return code:
```
- name: Fail task when both files are identical
  raw: diff foo/file1 bar/file2
  register: diff_cmd
  failed_when: diff_cmd.rc == 0 or diff_cmd.rc >= 2
```

You can also combine multiple conditions for failure. This task will fail if both conditions are true:
```
- name: Check if a file exists in temp and fail task if it does
  command: ls /tmp/this_should_not_be_here
  register: result
  failed_when:
    - result.rc == 0
    - '"No such" not in result.stdout'
```
If you want the task to fail when only one condition is satisfied, change the `failed_when` definition to:
```
failed_when: result.rc == 0 or "No such" not in result.stdout
```
If you have too many conditions to fit neatly into one line, you can split it into a multi-line yaml value with ` > `:
```
- name: example of many failed_when conditions with OR
  shell: "./myBinary"
  register: ret
  failed_when: >
    ("No such file or directory" in ret.stdout) or
    (ret.stderr != '') or
    (ret.rc == 10)
```
#### Overriding The Changed Result

When a shell/command or other module runs it will typically report "changed" status based on whether it thinks it affected machine state.

Sometimes you will know, based on the return code or output that it did not make any changes, and wish to override the "changed" result such that it does not appear in report output or does not cause handlers to fire:

```
tasks:

  - shell: /usr/bin/billybass --mode="take me to the river"
    register: bass_result
    changed_when: "bass_result.rc != 2"

  #### this will never report 'changed' status
  - shell: wall 'beep'
    changed_when: False
```
You can also combine multiple conditions to override "changed" result:
```
- command: /bin/fake_command
  register: result
  ignore_errors: True
  changed_when:
    - '"ERROR" in result.stderr'
    - result.rc == 2
```
#### Aborting the play

Sometimes it’s desirable to abort the entire play on failure, not just skip remaining tasks for a host.

The any_errors_fatal option will end the play and prevent any subsequent plays from running. When an error is encountered, all hosts in the current batch are given the opportunity to finish the fatal task and then the execution of the play stops. any_errors_fatal can be set at the play or block level:
```
- hosts: somehosts
  any_errors_fatal: true
  roles:
    - myrole
```
```
- hosts: somehosts
  tasks:
    - block:
        - include_tasks: mytasks.yml
      any_errors_fatal: true
```
for finer-grained control `max_fail_percentage` can be used to abort the run after a given percentage of hosts has failed

#### Blocks

Blocks allow for logical grouping of tasks and in play error handling. Most of what you can apply to a single task (with the exception of loops) can be applied at the block level, which also makes it much easier to set data or directives common to the tasks. This does not mean the directive affects the block itself, but is inherited by the tasks enclosed by a block. i.e. a when will be applied to the tasks, not the block itself.
Block example with named tasks inside the block

```
 tasks:
   - name: Install, configure, and start Apache
     block:
       - name: install httpd and memcached
         yum:
           name:
           - httpd
           - memcached
           state: present

       - name: apply the foo config template
         template:
           src: templates/src.j2
           dest: /etc/foo.conf
       - name: start service bar and enable it
         service:
           name: bar
           state: started
           enabled: True
     when: ansible_facts['distribution'] == 'CentOS'
     become: true
     become_user: root
     ignore_errors: yes
```

In the example above, each of the 3 tasks will be executed after appending the when condition from the block and evaluating it in the task’s context. Also they inherit the privilege escalation directives enabling “become to root” for all the enclosed tasks. Finally, ignore_errors: yes will continue executing the playbook even if some of the tasks fail.

Names for tasks within blocks have been available since Ansible 2.3. We recommend using names in all tasks, within blocks or elsewhere, for better visibility into the tasks being executed when you run the playbook.
Blocks error handling

Blocks also introduce the ability to handle errors in a way similar to exceptions in most programming languages. Blocks only deal with ‘failed’ status of a task. A bad task definition or an unreachable host are not 'rescuable' errors.
Block error handling example

```
 tasks:
 - name: Handle the error
   block:
     - debug:
         msg: 'I execute normally'
     - name: i force a failure
       command: /bin/false
     - debug:
         msg: 'I never execute, due to the above task failing, :-('
   rescue:
     - debug:
         msg: 'I caught an error, can do stuff here to fix it, :-)'
```

This will 'revert' the failed status of the task for the run and the play will continue as if it had succeeded.

There is also an `always` section, that will run no matter what the task status is.
Block with `always` section

```
 - name: Always do X
   block:
     - debug:
         msg: 'I execute normally'
     - name: i force a failure
       command: /bin/false
     - debug:
         msg: 'I never execute :-('
   always:
     - debug:
         msg: "This always executes, :-)"
```

They can be added all together to do complex error handling.
Block with all sections

```
- name: Attempt and graceful roll back demo
  block:
    - debug:
        msg: 'I execute normally'
    - name: i force a failure
      command: /bin/false
    - debug:
        msg: 'I never execute, due to the above task failing, :-('
  rescue:
    - debug:
        msg: 'I caught an error'
    - name: i force a failure in middle of recovery! >:-)
      command: /bin/false
    - debug:
        msg: 'I also never execute :-('
  always:
    - debug:
        msg: "This always executes"
```

The tasks in the block would execute normally, if there is any error the rescue section would get executed with whatever you need to do to recover from the previous error. The always section runs no matter what previous error did or did not occur in the block and rescue sections. It should be noted that the play continues if a rescue section completes successfully as it 'erases' the error status (but not the reporting), this means it won't trigger max_fail_percentage nor any_errors_fatal configurations but will appear in the playbook statistics.

Another example is how to run handlers after an error occurred :
Block run handlers in error handling
```
 tasks:
   - name: Attempt and graceful roll back demo
     block:
       - debug:
           msg: 'I execute normally'
         changed_when: yes
         notify: run me even after an error
       - command: /bin/false
     rescue:
       - name: make sure all handlers run
         meta: flush_handlers
 handlers:
    - name: run me even after an error
      debug:
        msg: 'This handler runs even on error'
```

Ansible also provides a couple of variables for tasks in the rescue portion of a block:

`ansible_failed_task` ->

The task that returned 'failed' and triggered the rescue. For example, to get the name use `ansible_failed_task.name`.

`ansible_failed_result` -> 

The captured return result of the failed task that triggered the rescue. This would equate to having used this var in the register keyword. 


## 9. Commonly Used Files Modules with Examples

The file module acts like `chcon` when setting file contexts:
```
- name: Create a file
  hosts: localhost
  become: true
  tasks:
    - name: Create a file and set permissions
      file:
        path: /root/file
        owner: root
        group: root
        mode: 0640
        state: touch
        setype: user_tmp_t
    - name: SELinux type is persistently set to user_tmp_t
      sefcontext:
        target: /root/file
        setype: user_tmp_t
        state: present
```
```
- name: Copy a file
  hosts: localhost
  become: true
  tasks:
    - name: Copy a file
      copy:
        src: file
        dest: /root/file
        force: yes
        owner: root
        group: ansible
        mode: 0640
```
```
- name: Add a line to a file
  hosts: localhost
  become: true
  tasks:
    - name: Add a new line to a file
      lineinfile:
        path: /root/file
        line: "this is the new line"
        state: present
```
```
- name: Add block of text to a file
  hosts: localhost
  become: true
  tasks:
    - name: Add a block of text to an existing file
      blockinfile:
        path: /root/file
        block: |
          This is the first line to be added.
          This is the second line to be added.
        state: present
```
```
- name: Download a file from managed hosts
  hosts: localhost
  become: false
  tasks:
    - name: Fetch a file
      fetch:
        src: /etc/hosts
        dest: my-folder
        flat: no
```
```
- name: Download a file from remote host
  hosts: localhost
  become: false
    - name: Download file from URL
      get_url:
        url: http://katello.hl.local/pub/index.html
        dest: /tmp/index.html
```
```
- name: Disable SSH root login
  hosts: localhost
  become: true
  tasks:
  - name: Modify SSH server config
    lineinfile:
      dest: "/etc/ssh/sshd_config"
      regexp: "^PermitRootLogin"
      line: "PermitRootLogin no"
```
## 10. Password Hashing

How to generate SHA512 crypted passwords for the user module? The answer is taken from Ansible FAQ.
```
- name: Create user with password
    user:
      name: sandy
      password: "{{ user_pw | password_hash('sha512') }}"
```

## 11. Jinja2 Templates

Similar to Puppet. Puppet templates are based upon Ruby's ERB, Ansible templates are based upon Jinja2.

A file containing a Jinja2 template does not need to have any specific file extension.

Use the template module to deploy it:
```
- name: Use a template file
  hosts: localhost
  gather_facts: yes
  tasks:
    - name: Deploy index.html
      template:
        src: templates/index.j2
        dest: /var/www/html/index.html
```
```
$ cat templates/index.j2
The system kernel is: {{ ansible_kernel }}
```

## 12. Ansible Roles

Similar to Puppet modules, reusable code in a modular fashion. Create a role skeleton:
```
$ mkdir roles && cd roles
$ ansible-galaxy init my_test_role
```
```
$ tree my_test_role/
my_test_role/
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```
As of Ansible 2.4, you can use roles inline with any other tasks using `import_role` or `include_role`:
```
---
- hosts: webservers
  tasks:
  - debug:
      msg: "before we run our role"
  - import_role:
      name: example
  - include_role:
      name: example
  - debug:
      msg: "after we ran our role"
```
Search for roles from the CLI:
```
$ ansible-galaxy search --author redhat
$ ansible-galaxy search --galaxy-tags tomcat
$ ansible-galaxy search tomcat --platforms EL
```
Ansible Galaxy is a public library of Ansible roles written by users. Similar to Puppet Forge. Install a role from Galaxy:
```
$ ansible-galaxy install <author.name> -p ~/.ansible/roles
```
List installed roles:
```
$ ansible-galaxy list -p ~/.ansible/roles/
- <author.name>, 1.9.5
```
To define role source, use `requirements.yml`:
```
- src: http://katello.hl.local/pub/ansible-haproxy.tar.gz
  name: haproxy
- src: http://katello.hl.local/pub/ansible-mysql.tar.gz
  name: mysql
```
```
$ ansible-galaxy init -r requirements.yml
```
The original way to use roles is via the roles: option for a play:
```
---
- hosts: webservers
  roles:
     - webservers
```
The order of execution for your playbook is as follows:

1. Any `pre_tasks` defined in the play.
2. Any handlers triggered so far will be run.
3. Each role listed in roles will execute in turn (with dependencies).
4. Any `tasks` defined in the play.
5. Any handlers triggered so far will be run.
6. Any `post_tasks` defined in the play.
7. Any handlers triggered so far will be run.

Role dependencies are always executed before the role that includes them, and may be recursive.

### RHEL System Roles
```
$ sudo yum install rhel-system-roles
$ ls /usr/share/doc/rhel-system-roles-1.0/
kdump
network
postfix
selinux
timesync
```
```
$ ansible-galaxy list
- rhel-system-roles.selinux, (unknown version)
- linux-system-roles.network, (unknown version)
- rhel-system-roles.postfix, (unknown version)
- rhel-system-roles.timesync, (unknown version)
- rhel-system-roles.kdump, (unknown version)
- linux-system-roles.timesync, (unknown version)
- linux-system-roles.kdump, (unknown version)
- linux-system-roles.postfix, (unknown version)
- rhel-system-roles.network, (unknown version)
- linux-system-roles.selinux, (unknown version)
```

## 13. Ansible Variables

Variables for hosts and host groups can be defined by creating two directories, `group_vars` and `host_vars`, in the same working directory as the inventory file or directory.
```
$ cat group_vars/webserver
my_package: httpd
```
```
$ cat group_vars/database
my_package: mariadb
```
Whether or not you define any variables, you can access information about your hosts with the Special Variables Ansible provides, including "magic" variables, facts, and connection variables.

The most commonly used magic variables are:
```
hostvars
groups
group_names
inventory_hostname
```
The `group_names` variable contains a list of all the groups the current host is in. We can use it to install specific packages:
```
tasks:
  - name: Install webserver packages
    package:
      name: "{{ item }}"
      state: latest
    loop:
      - httpd
      - firewalld
    when: "'webserver' in group_names"
```
The `hostvars` variable lets you access variables for another host, including facts that have been gathered about that host.
```
{{ hostvars['ansible2.hl.local']['ansible_default_ipv4']['address'] }}
{{ hostvars['ansible2.hl.local']['ansible_fqdn'] }}
```
The `inventory_hostname` variable is the name of the hostname as configured in Ansible's inventory host file. The groups variable is a list of all the groups in the inventory.
```
when: inventory_hostname in groups["webservers"]
```

## 14. meta - Execute Ansible Actions

Meta tasks are a special kind of task which can influence Ansible internal execution or state. Choices:
```
* flush_handlers
* refresh_inventory
* noop
* clear_facts
* clear_host_errors
* end_play
* reset_connection
```
Example meta task for how to run handlers after an error occurred:
```
 tasks:
   - name: Attempt and graceful roll back demo
     block:
       - debug:
           msg: 'I execute normally'
         notify: run me even after an error
       - command: /bin/false
     rescue:
       - name: make sure all handlers run
         meta: flush_handlers
 handlers:
    - name: run me even after an error
      debug:
        msg: 'This handler runs even on error'
```
Note that meta is not really a module as such it cannot be overwritten.

## 15. Ansible Check Mode (Dry Run)

When `ansible-playbook` is executed with `--check` it will not make any changes on remote systems. To modify the check mode behavior of individual tasks, you can use the check_mode option:
```
- name: this task will run under checkmode and not change the system
  lineinfile:
    line: "PermitRootLogin no"
    dest: /etc/ssh/sshd_config
    state: present
  check_mode: yes
```
The above will force a task to run in check mode, even when the playbook is called without `--check`.

## 16. Ansible Playbook Debugger

Ansible includes a debugger as part of the strategy plugins. This debugger enables you to debug as task.

The debugger keyword can be used on any block where you provide a name attribute, such as a play, role, block or task.
```
---
- hosts: ansible2.hl.local
  tasks:
    - unarchive:
        src: files/archive.tgz
        dest: /tmp/
        remote_src: no
    - name: archive file
      debugger: on_failed
      archive:
        path: /tmp/archive.html
        dest: /tmp/file.gz
        format: tgz
```

## 17. Ansible Ignoring Failed Commands

Playbooks will stop executing any more steps on a host that has a task fail. To ignore this behaviour, set `ignore_errors` to `true`:
```
- name: this will not be counted as a failure
  command: /bin/false
  ignore_errors: yes
```
This is useful when you expect a task to fail. For example, you are checking if Apache website is reachable. It may be unreachable, but you don't want the play to fail because of that.

## 18. Ansible Run Once

There may be a need to only run a task one time for a batch of hosts. This can be achieved by configuring `run_once` on a task:
```
- command: /tmp/upgrade_database.sh
  run_once: true
```

## 19. Ansible Aborting the Play

There will be cases when you will need to abort the entire play on failure, not just skip remaining tasks for a host, to avoid breaking the system. The `any_errors_fatal` play option will mark all hosts as failed if any fails, causing an immediate abort:
```
- hosts: all
  any_errors_fatal: true
  roles:
    - database
```

## 20. Ansible Keywords

These are some of the keywords available on common playbook objects.

Notable play keywords:
```
any_errors_fatal
become
become_method
become_user
check_mode
debugger
force_handlers
gather_facts
handlers
hosts
ignore_errors
name
no_log
order
port
post_tasks
pre_tasks
remote_user
roles
run_once
serial
tasks
vars
vars_files
```
Notable block keywords:
```
always
block
rescue
when
```
Notable task keywords:
```
loop
register
```

## 21. Ansible Setting Defaults for Modules

It can be useful to define default arguments for a particular module using the `module_defaults` attribute:
```
- hosts: all
  become: true
  module_defaults:
    file:
      owner: root
      group: devops
      mode: "664"
      state: touch
  tasks:
    - file:
        path: /tmp/file1
    - file:
        path: /tmp/file2
    - file:
        path: /tmp/file3
```
It is a time saver when you need to use the same module repeatedly.

## 23. Ansible - WAIT_FOR and WAIT_FOR_CONNECTION

#### wait_for – Waits for a condition before continuing

Examples:

```
- name: sleep for 300 seconds and continue with play
  wait_for:
    timeout: 300
  delegate_to: localhost

- name: Wait for port 8000 to become open on the host, don't start checking for 10 seconds
  wait_for:
    port: 8000
    delay: 10

- name: Waits for port 8000 of any IP to close active connections, don't start checking for 10 seconds
  wait_for:
    host: 0.0.0.0
    port: 8000
    delay: 10
    state: drained

- name: Wait for port 8000 of any IP to close active connections, ignoring connections for specified hosts
  wait_for:
    host: 0.0.0.0
    port: 8000
    state: drained
    exclude_hosts: 10.2.1.2,10.2.1.3

- name: Wait until the file /tmp/foo is present before continuing
  wait_for:
    path: /tmp/foo

- name: Wait until the string "completed" is in the file /tmp/foo before continuing
  wait_for:
    path: /tmp/foo
    search_regex: completed

- name: Wait until regex pattern matches in the file /tmp/foo and print the matched group
  wait_for:
    path: /tmp/foo
    search_regex: completed (?P<task>\w+)
  register: waitfor
- debug:
    msg: Completed {{ waitfor['groupdict']['task'] }}

- name: Wait until the lock file is removed
  wait_for:
    path: /var/lock/file.lock
    state: absent
```

#### wait_for_connection – Waits until remote system is reachable/usable

```
- name: Wait 600 seconds for target connection to become reachable/usable
  wait_for_connection:

- name: Wait 300 seconds, but only start checking after 60 seconds
  wait_for_connection:
    delay: 60
    timeout: 300
```


## 24. Ansible - Spanning multiple lines

Values can span multiple lines using | or >. Spanning multiple lines using a "Literal Block Scalar" ` | ` will include the newlines and any trailing spaces. Using a "Folded Block Scalar" ` > ` will fold newlines to spaces; it’s used to make what would otherwise be a very long line easier to read and edit. In either case the indentation will be ignored. Examples are:

```
include_newlines: |
            exactly as you see
            will appear these three
            lines of poetry

fold_newlines: >
            this is really a
            single line of text
            despite appearances
```

## 25. Ansible Best Practices

Tips for making the most of Ansible and Ansible playbooks.

1. Always mention the state. The `state` parameter is optional to a lot of modules. Whether `state=present` or `state=absent`, it is always best to leave that parameter in your playbooks to make it clear.
2. Generous use of whitespace to break things up, and use of comments, which start with `#`, is encouraged.
3. Always name tasks. It is recommended to provide a description about why something is being done!
4. Keep it simple. Do not attemtp to use every feature of Ansible together, all at once. Use what works for you!
5. Ansible best practice has no limit on the amount of variable and vault files or their names.
