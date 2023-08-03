---
title: Authentication and Authorization using Azure Enterprise Applications
categories: [azure]
tags: [azure, authentication, authorization]
---

# Ansible - Roles

This is a series where I talk about how I like to organize my Ansible code.

Ansible - Setup \[Coming Soon\]
[Ansible - Basics]({% post_url 2023-08-02-ansible-basics %})
[Ansible - Roles]({% post_url 2023-08-03-ansible-roles %}) <- (you are here)
Ansible - Variables \[Coming Soon\]
Ansible - Templating \[Coming Soon\]
Ansible - Pull \[Coming Soon\]

## Roles

Roles are a way to break up and organize tasks.
You can think of roles like Legos, where each one does something on its own,
but can also be composed with other roles to fully configure a system.

For example, a simple web server, `quackweb`, might include these roles:

- ansible
    - configures the ansible service account
    - ansible-pull, in a later lesson.
- auth
    - configures sssd and pam
    - allows access for your admins
- firewall
    - configures nftables
    - uses variables to dynamically set rules
- web
    - installs and configures apache service
    - ensures web content
- quackweb
    - any unique configuration for quackweb that nothing else will ever use


## Role Components

Roles live in the `roles` directory at the root of your ansible repository:

```
ansible-repository
├── ansible.cfg
├── .vault-pass
├── playbooks
├── roles
│   ├── web                 <- these
│   ├── some_other_role1    <- are
│   ├── some_other_role2    <- directories
│   ├── web_proxy.yml
│   └── quackweb.yml
```

You'll see that there's both a `roles` directory, and a `playbooks` directory.
There is a simple, yet important distinction to make here:

The `roles` directory, and everything inside of it, is ansible code that *is
always safe to run over and over, as many times as possible*. Roles should be
idempotent. Roles are what you would consider "configuration management", and 
should be 100% okay to have the server pull its own role every hour,
every night, or any time.

The `playbooks` directory is for one-off tasks, such as patching servers,
performing one-off backups, etc. The `playbooks` directory is more "wild west",
and you can feel free to organize that directory however you'd like.

Since this lesson is for Roles, let's focus on those. The directory structure 
for a single role looks like this:

```
web
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── tasks
│   ├── component1.yml
│   ├── component2.yml
│   └── main.yml
└── templates
```

For the tasks, handlers, and defaults directories, you'll notice that there's 
a `main.yml` file. That's the entry point for ansible when it tries to load 
the role.


### tasks

The tasks directory is where almost all of the ansible code goes.
How you organize the files in this directory is up to you (whether
you put everything inside `main.yml` or break it up into components). 

I prefer to break my tasks directory up into components, and use the `main.yml`
file as a thin loader for the components, adding tags to make running 
individual components easier.

For the example `web` role above, my tasks directory might include the
following files:

```
main.yml    
configure_httpd.yml     <- install httpd, write ssl.conf and cert/key
ensure_content.yml      <- write your html, deploy your app, whatever
```

The contents of these files is simply a list of tasks. You don't specify hosts
or any connection information, as that comes from the group playbook that 
we'll get to later. Given the example files above, `main.yml` would look like:

```yaml
---
- name: configure_httpd
  ansible.builtin.import_tasks: configure_httpd.yml
  tags:
    - web
    - web:configure_httpd

- name: ensure_content
  ansible.builtin.import_tasks: ensure_content.yml
  tags:
    - web
    - web:ensure_content
```

As you add more components to the role, you'll just add more blocks into 
`main.yml` to load and tag those components.

Like all tasks, each one of these starts with a `name` property. I like to 
make the `name` exactly match the filename of the component. Next, you use 
the `import_tasks` module, which looks inside the current directory and tries
to load the file specified. Last, the `tags` property allows you to specify
any number of tags for the task. We'll come back to this, but examine
the structure of how these tags are configured. It matters!

If you were wondering, the content of `configure_httpd.yml` might look like:

```yaml
---
- name: install httpd and mod_ssl
  ansible.builtin.package:
    state: present
    name: 
      - httpd
      - mod_ssl

# more tasks defined below, like configuring SSL
```


### handlers

The `handlers` directory is very similar to the `tasks` directory, except
I usually just store all my handlers directly in the `main.yml` file:

```yaml
---
- name: restart_httpd
  ansible.builtin.service:
    name: httpd
    state: restarted
    enabled: true

- name: restart_tomcat
  ansible.builtin.service:
    name: tomcat
    state: restarted
    enabled: true
```

As long as your handlers are defined in the `handlers/main.yml` file, you can 
use the `notify` property on your tasks just like normal.


### files and templates

The `files` and `templates` directories are pretty straightforward.

- `files`: This is where the `copy` module will look for files.
- `templates`: This is where the `template` module will look for templates.

The real can-of-worms gets opened when I make the statement:

> Most configurations in a role should be a template, and use variables
    to interpolate *any* configurable options into the template. Files should 
    only be used when there should be *no* difference between using this role
    from one server to the next.

In reality, it's fuzzy. I try to use templates more than files, especially
when I'm planning on using the role on multiple servers. If the role is really
just for one server, I'll name it accordingly, (`quackhost`), and I can put
all the custom configuration I want in there, because I know I'll never apply
the `quackhost` role to my Satellite server.

For templates, I prefer to suffix all the files with the `.j2` extension 
(`ssl.conf.j2`), as templates use the `jinja2` templating system. It helps for 
when I'm searching for files in my ansible repository and I can see, just 
based on the filename, if a file is a template or a static file.


### defaults

The `defaults` directory is usually just a single `main.yml` file that stores
all the configurable variables defined in the role. That way, if my future-self
asks, "What variable determines the listening port for tomcat again?", I can
simply look in the `defaults/main.yml` file and I'll see something like this:

```yaml
---
web_tomcat_listen_host: "0.0.0.0"
web_tomcat_listen_port: "8000"
web_tomcat_site_base_directory: ""
```

So I know to search my ansible repository for the string 
`web_tomcat_listen_port` and I should find it configured in the `host_vars`
or `group_vars` inventory directories. More on that later!

You'll also notice that some of the values for these variables are empty. 
The rule of thumb is that you *only* want to store the values in this file 
if you could hypothetically hand this role to some random person and the
value of that property would be useful to them as well.

In this example, the `web_tomcat_listen_host` and `web_tomcat_listen_port` are
pretty universal values, and would generally work if someone just picked this
up and ran with it. However, the `web_tomcat_site_base_directory` is probably
specific to whoever is configuring the server. Should it go in `/var/www/html`?
Maybe `/opt/site`, or `/srv/`? Our answer won't necessarily match everyone 
else's answer, so we leave this one blank. That way, if someone else (or 
ourselves) uses this module without configuring it correctly, ansible will fail
and give a nice error message that this variable is unset (sometimes...).

The last thing to notice is that we *always* prefix the variable name with the
role name, in this case `web`. This is important, as ansible variables are all
in the same namespace, and this will prevent collisions. For example, if you 
had two roles that both used the variable `listen_port`, then both roles
would use the value of `listen_port` from the role loaded last.


## Composing Roles into the Group Playbook

Now that we understand how a single role is configured, we can look at the 
"Group Playbook" that ties the room together. 

The Group Playbook is a 1-to-1 playbook to a group. This group is configured in 
your `inventory.yml`, and may be comprised of one or many servers, though in our 
environment, usually it's a single relationship.

`group -> server1`: Some unique server is the only member of a group, and that 
    server provides bespoke functionality.

`group -> server1,server2`: In this case, `server1` and `server2` 
    *should be identical!*

The content of the the `quackweb.yml` group playbook looks like this:

```yaml
---
# You should put some documentation in the comments here at the top.
# Things like contacts, purpose, etc. Once you have all your servers
# documentated in roles, there's really no reason for confluence documents!
#
- name: quackweb
  hosts: quackweb
  become: true

  vars_files:
    - ../inventory/group_vars/all.yml
    - ../inventory/group_vars/quackweb.yml

  tasks:
    - name: roles
      block:
        - name: ansible
          ansible.builtin.import_role:
            name: ansible

        - name: auth
          ansible.builtin.import_role:
            name: auth

        - name: firewall
          ansible.builtin.import_role:
            name: firewall

        - name: web
          ansible.builtin.import_role:
            name: web

        - name: quackweb
          ansible.builtin.import_role:
            name: quackweb
```

You might notice that this looks pretty similar to any normal playbook. It is!
Everything in ansible is some form of playbook, play, or task, and the `roles`
schema is just a good way to organize your infrastructure-as-code.

Starting from the top:

`name`: Just make it match the group.
`hosts`: The group name. Should match the role name.
`become`: We connect using our ansible service account, but we want
    that account to elevate to root to run all this configuration.
`vars_files`: Ansible has many ways of loading variables, and I don't want to 
    mentally keep track of all these locations, so I explicitly load variable 
    files from the `inventory` directory. We'll go over role variables in
    detail in the next lesson.
`tasks`: This is how we start to load the roles.

The `tasks` property just has a single meta task, the `block` module. This can
be used to do some better error handling, which we'll look at later. Inside
the `block` module, we have our list of tasks, which are all `import_role`
tasks. This is how each role is loaded. Keep in mind that these are loaded 
and run *in order*, so make sure you have them ordered correctly!

All that's left is to run it:

`ansible-playbook roles/quackweb.yml`

That's it! All your code will run in order, targeting the correct group.

Earlier we talked about tags on our role components. Here's some additional
information about those tags now that we understand the whole stack.


### Tagging!

Tags can make your life easier during development and troubleshooting. They 
allow you to run *only* tasks that are tagged with the tags you specify when 
running the playbook. 

In the example above, the `configure_httpd` task imports one or many other 
tasks from the `configure_httpd.yml` file. By tagging the block in the 
`main.yml` file, the tags will be applied to *all* tasks in that 
component file. 

By convention, we specify two tasks for each component in `main.yml`:

- The name of the role, in this case `web`
- The name of the role, then the name of the component (`configure_httpd`), 
    separated by a colon `:`.

When we run the group playbook, (explained in more detail later), we can 
optionally specify any number of tags with the `-t` command flag.
By tagging each of these blocks with both `web` and `web:<component>`, we can 
run the group playbook several ways:

`ansible-playbook quackweb.yml`: Runs the group playbook normally, 
    running all roles in order.
`ansible-playbook quackweb.yml -t web`: Runs *all* components in the 
    `web` role. 
`ansible-playbook quackweb.yml -t web:configure_httpd`: Runs *only* the
    `configure_httpd` component in the `web` role.

I find this very useful when working on new ansible code, as I can just run
the group playbook with the tag of the role/component I'm currently
working on. It definitely speeds up development!


