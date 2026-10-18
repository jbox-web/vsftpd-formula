# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This formula follows the `template-formula` layout (`map.jinja` merge chain,
`package`/`config`/`service` split, Test Kitchen + InSpec). Only what is specific to
`vsftpd-formula` is written here.

## Commands

```bash
bundle install
bin/kitchen list                                 # 3 instances, suite `default` only
bin/kitchen test default-debian-13-3008-2        # destroy + converge + verify + destroy
bin/kitchen verify default-debian-11-3006-27     # InSpec controls on a converged instance
pre-commit run --all-files                       # rubocop and yamllint lint the whole tree
```

The platform names in `docs/README.rst` (`debian-9-2019-2-py3`) are upstream leftovers;
`kitchen.yml` is the source of truth.

## Two pillar namespaces, not one

The formula reads configuration from two unrelated places:

- `vsftpd:` (and `vsftpd:lookup`), merged by `vsftpd/map.jinja` over `defaults.yaml` and
  the os maps. It only carries packaging facts: `server` (package name), `service`,
  `vsftpd_config` (path of the rendered file) and `vsftpd_config_src`. Only
  `osfamilymap.yaml` has content (FreeBSD, RedHat, Gentoo); Debian uses the defaults.
- `vsftpd_config:`, a **top-level** pillar key read directly with `pillar.get`, bypassing
  `map.jinja` entirely. It holds the vsftpd options themselves. `pillar.example` and
  `test/salt/pillar/vsftpd.sls` only set this key.

`vsftpd/config/file.sls` dereferences `vsftpd_config.secure_chroot_dir` with no default,
so the pillar must define it, and adds a `file.exists` check when `anon_root` is set.

## The `vsftpd.conf` template

`vsftpd/files/vsftpd.conf` is the only template (no `libtofs.jinja`, `source:` points
straight at it). Its macros decide how each option is rendered:

- `option(key, default)`: emitted commented out (`#key=default`) unless `key` is in
  `vsftpd_config`, in which case the pillar value is emitted.
- `option_default_uncommented(key, default)`: always emitted, pillar value or default.
- Booleans become `YES`/`NO`; a list emits one `key=value` line per item.
- Pillar keys the template does not list are appended at the end of the file, so any
  vsftpd option can be set without touching the template.

`chroot_local_user` is listed twice in the template; when the pillar sets it, it is
rendered twice.

## State chain

`init.sls` → `package.install` (`pkg.installed`) → `config.file` (config file + chroot
directory; includes `package.install` but declares no `require` on it) →
`service.running` (enabled, `watch` on
`config.file`, so any config change restarts vsftpd; there is no `check_cmd`).

## Tests

- One suite, `default`; its InSpec controls check the package, the service, port 21
  listening, and four lines of the rendered config (`config_spec.rb`).
- There are no `_mapdata` reference dumps: `vsftpd._mapdata` runs in `state_top` but no
  control compares its output. A change to `defaults.yaml` is therefore untested.
