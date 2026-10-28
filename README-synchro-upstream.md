# Synchronisation Upstream - Fork MySQL

## Pourquoi ce fork existe

Le role upstream `geerlingguy/ansible-role-mysql` 6.3.1 :

- force `/var/log/mysql` en `0755` et ecrase le `02750` du paquet Debian (setgid, groupe `adm`)
- relance le secure-install des qu'un upgrade de paquet rend `mysql_install_packages` vrai : `ALTER USER` vers `unix_socket`, reecriture de `/root/.my.cnf`, suppression des comptes `root` distants, `DROP DATABASE test`

Il n'y a pas de variable upstream pour ces deux comportements.

## Perimetre du fork

- Upstream source: `https://github.com/geerlingguy/ansible-role-mysql`
- Fork source: `https://github.com/CytadelHosting/ansible-role-mysql`
- Collection consommatrice: `cytadel.webserver` (`roles.lock.yaml`, `role_name: geerlingguy_mysql`)

Patches a conserver :

- `mysql_log_dir_mode` / `__mysql_log_dir_mode: "02750"` sur Debian
- `mysql_secure_installation` (null = comportement upstream, `false` = ne rien muter)

## Remotes

```bash
git remote -v
# origin    git@github.com:CytadelHosting/ansible-role-mysql.git
# upstream  https://github.com/geerlingguy/ansible-role-mysql.git
```

- `master` suit `upstream/master`
- `cytadel-release` est la branche poussee sur `origin` et referencee par la collection

## Strategie de synchronisation

```bash
git fetch upstream --tags
git fetch origin
git checkout cytadel-release
git merge --no-ff 6.3.1
```

Apres le merge, verifier :

- `tasks/configure.yml` utilise `mysql_log_dir_mode`
- `tasks/secure-installation.yml` est gate par `mysql_secure_installation`
- `vars/Debian*.yml` contient `__mysql_log_dir_mode: "02750"`

## Convention de tagging

`v<UPSTREAM>-cytadel.<N>`

Le compteur repart a 1 quand la base upstream change.

- `v6.3.1-cytadel.1` premiere release sur le tag upstream `6.3.1`
- `v6.3.1-cytadel.2` deuxieme patch sur la meme base
- `v6.4.0-cytadel.1` prochaine base upstream

```bash
git tag -a v6.3.1-cytadel.1 -m "Fork release based on upstream 6.3.1 (Cytadel patchset #1)"
git push -u origin cytadel-release
git push origin master
git push origin v6.3.1-cytadel.1
```

## Integration dans la collection

`roles.lock.yaml` :

```yaml
- role_name: "geerlingguy_mysql"
  src: "git@github.com:CytadelHosting/ansible-role-mysql.git"
  version: "v6.3.1-cytadel.1"
```

Le `role_name` ne change pas : les playbooks continuent d'appeler `cytadel.webserver.geerlingguy_mysql`.

```bash
make -f scripts/Makefile validate-lock
make -f scripts/Makefile build
```
