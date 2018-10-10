Ansible Role: satellite
=======================

This role will configure a system (at this point a VM on a laptop...) to be used
as a Red Hat Satellite server.  It is loosely based on the
[Satellite Quick Start Guide](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.3/html-single/quick_start_guide).

It is (and probably always will be) a work in progress.  There are various
other approaches to this, and regarding the layout of this role, I made the
following decisions:

- this is a single role, but follows a heirarchy that I attempted to define in
  the naming of the tasks (these are likely to change as I add functionality
  and I'm unlikely to remember to update this...):
  
        $ tree roles/satellite/tasks
        roles/satellite/tasks
        ├── 01_os_config.yml
        ├── 02_app_install.yml
        ├── 03_env_config.yml
        ├── 03a_manifest.yml
        ├── 03b_sat_domain.yml
        ├── 03c_openscap.yml
        ├── 03d_sync_config.yml
        ├── 03e_lifecycles.yml
        ├── 03f_RHrepos.yml
        ├── 03g_nonRHproducts-repos.yml
        ├── 03h_standardCVs.yml
        ├── 03i_compositeCVs.yml
        ├── 03j_CVfilters.yml
        └── main.yml

  - this is so that the tasks can use the same variables
  - also hope that it will be easier to go back and idempotentize (?) the tasks
- tags are a bit messed up
- I have attempted to keep from hardcoding anything in the tasks and rely on
  variables that will be passed to the role (in any number of ways)
  - this creates quite a nightmare in role invocation and/or a variables file
    that is very specific to this role.  I've tried to document what each task
    uses in the top of each file
- hopefully this means I have made this generic enough that the larger public
  can use it

The result is a Satellite ready for reproductions and demonstrations.

Important Notes
---------------

- This is currently developed to be used on a system locally.  In order to use
  it with Tower or AWX will take some refactoring
- This role is far from idempotent at this point.  During testing I have found
  the following commands helpful to clean up:

      sudo virsh destroy <vm.name>;
      sudo virsh undefine <vm.name> --remove-all-storage

- Please refer to the *(almost)* working example

Requirements
------------

I couldn't escape a few requirements:

- You'll need to have your libvirt environment configured to your liking.  I'm
  working on creating my own preferences in a separate role...to be continued...
- You need to have an appropriate subscription for Satellite packages and
  unfortunately I currently have not been able to get an activation key so there
  is a step that waits for you to register the new system (dangit)
- You need to download **manifest** and name it as `/tmp/manifest.zip` on the
  system invoking the role/playbook (there is a task to wait for this)

Role Variables
--------------

All of these variables should be considered **required** however, it will
depend greatly on your own situation and configuration.  Just know that there is
currently no sanity checking (and I'm insane):

- `sat` - I've created a bit of a nested list here so that the variables can be 
  used like sat.organization
  - `admin_password` - this is the `admin` password for use on the webUI
  - `organization` - the organization which will be created at install and used
    throughtout
  - `location` - another aspect of permission granularity, created at install
  - `dhcp_range` - since we will configure the Satellite to serve DHCP (for
    provisioning) we need to specify a range
- `lifecycles` - allows the definition of multiple lifecycles *(see example 
  below for usage)*
  - `name` - the name of a lifecycle (a lifecycle environment will be created by
    default to accomodate this)
  - `prior` - we need to define the order so include the prior lifecycle here
- `RHrepos` - these are repositories provided by RH that just need to be enabled
  *(see example below for usage)* *see the notes in the RHrepos task*
  - `set_name` - the name of repository needed for the `repository-set` command
  - `repo_name` - the name needed for other `repository` commands
  - `label` - unused at time of writing
  - `basearch` - the architecture if the repo is arch specific
  - `releasever` - the version of the repo to tie to (***optional***)
- `nonRHproduct` - these are 3rd party repos so a lot more info is needed
  - `name` - what you want to name the **product**
  - `gpg_key_url` - the public GPG key for the packages comprising this
    repo/product
  - `repo` - this is information about the **repo**(s) that in the product
    - `name` - this is what you want to name it
    - `url` - this will correspond to `baseurl`
    - `type` - what type of repo this will be (probably `yum`)
    - `http` - specify if the content should be pubished to http
    - `chksum` - the checksum algorithm used to verify
- `standardCVs` - this is a list of **non**-composite Content Views and the
  needed information about them
  - `name` - the name of your Content View
  - `desc` - a brief description of it
  - `prod` - this is the *product* that the repos are in (is this getting
    confusing)
  - `repos` - this is a ***list*** of repositories that will make up the content
    view

Example Playbook
----------------

Playbook with configuration options specified:

~~~yaml
- name: the VM is created, now configure it
  hosts: satVM
  remote_user: root
  vars:
    sat:
      admin_password: pASSw0rd
      organization: myOrg
      location: myLoc
      dhcp_range: "192.168.1.150 192.168.1.200"
    lifecycles:
      - name: Development
        prior: Library
      - name: Acceptance
        prior: Development
      - name: Production
        prior: Acceptance
      - name: Lab
        prior: Library
    RHrepos:
      - set_name: "Red Hat Enterprise Linux 7 Server (RPMs)"
        repo_name: "Red Hat Enterprise Linux 7 Server RPMs x86_64 7Server"
        label: "rhel-7-server-rpms"
        basearch: "x86_64"
        releasever: "7Server"
      - set_name: "Red Hat Enterprise Linux 7 Server (Kickstart)"
        repo_name: "Red Hat Enterprise Linux 7 Server Kickstart x86_64 7.5"
        basearch: "x86_64"
        releasever: "7.5"
      - set_name: "Red Hat Satellite Tools 6.3 (for RHEL 7 Server) (RPMs)"
        repo_name: "Red Hat Satellite Tools 6.3 for RHEL 7 Server RPMs x86_64"
        basearch: "x86_64"
    nonRHproduct:
      - name: "Extra Packages for Enterprise Linux"
        gpg_key_url: "https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7"
        repo:
          name: "EPEL 7 - x86_64"
          url: "http://dl.fedoraproject.org/pub/epel/7/x86_64/"
          type: "yum"
          http: "true"
          chksum: "sha256"
    standardCVs:
      - name: "RHEL7_Base"
        desc: "Core Build for RHEL7"
        prod: "Red Hat Enterprise Linux Server"
        repos:
          - "Red Hat Enterprise Linux 7 Server RPMs x86_64 7Server"
          - "Red Hat Enterprise Linux 7 Server Kickstart x86_64 7.5"
          - "Red Hat Satellite Tools 6.3 for RHEL 7 Server RPMs x86_64"
      - name: "EPEL7"
        desc: "Only Extra Packages for Enterprise Linux 7"
        prod: "Extra Packages for Enterprise Linux"
        repos:
          - "EPEL 7 - x86_64"
  roles:
    - satellite
~~~

Inclusion
---------

I envision this role being included in a larger project through the use of a `requirements.yml` file. So here is an example of what you would need:

~~~yaml
# get the satellite role from github
- src: https://github.com/thisdwhitley/ansible-role-satellite.git
  scm: git
  name: satellite
~~~

Having the above in a `requirements.yml` file in your project would allow you to "install" it (prior to use in some sort of setup script?) with:

~~~bash
ansible-galaxy install -p ./roles -r requirements.yml
~~~

Testing
-------

The complexity of this role does not lend itself to easy testing.  I have found
that I can create an inventory and then comment/uncomment what I want to do.  It
is kind of crappy, sorry about that.

But because of that...I'm not going to put this in galaxy.  Don't judge.

References
----------

- [rickmanley-nc's satellite repo](https://github.com/rickmanley-nc/satellite)
  *I borrowed heavily from Ricky's work!*

License
-------

Red Hat, the Shadowman logo, Ansible, and Ansible Tower are trademarks or
registered trademarks of Red Hat, Inc. or its subsidiaries in the United
States and other countries.

All other parts of this project are made available under the terms of the [MIT
License](LICENSE).
