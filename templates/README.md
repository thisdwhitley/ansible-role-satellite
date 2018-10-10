I feel a little weird about having this kickstart file in a role that does not
use it, but it defines a Satellite VM...

**Ideally** this `sat6-ks.cfg.j2` would be used in conjunction with my
[ansible-role-KSbuilder](https://github.com/thisdwhitley/ansible-role-KSbuilder)
role.  You'd just need to place this template in the `KSbuilder/templates/`
directory as outlined in that role.