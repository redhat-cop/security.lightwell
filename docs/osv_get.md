# Lightwell OSV playbook

`security.lightwell.osv_get` downloads a Lightwell Open Source Vulnerability (OSV) catalog,
compares it with a local reference, and prints the vulnerabilities that are new or whose file content has changed.

`lightwell_tech_osv` selects the technology. `java`, `python` and `javascript` are supported. The default is `java`.
`lightwell_type_osv` selects the type of fix. `remediated` and `predisclosure` are supported. The default is `remediated`.
The catalog is the remediated OSV repository for that technology on the Red Hat Pulp content API:

`https://packages.redhat.com/api/pulp-content/lightwell/osv/<technology>/<type>`

Manifest files are stored in a local directory named after the technology and type, e.g. `./java/predisclosure` or `./python/remediated`.
The play creates that directory when it is missing.

The play runs on `localhost`. It does not gather facts and does not escalate privileges.

## What the play does

1. Reads `LIGHTWELL_USER` and `LIGHTWELL_TOKEN` from the environment and stops if either one is empty. It also stops unless `lightwell_tech_osv` is `java` or `python`.
2. Creates `.../<technology>/<type>` and downloads `PULP_MANIFEST` into that directory with HTTP basic authentication. When `PULP_MANIFEST.ref` is missing, the play copies the manifest into that file. An existing reference is left as it is. A `304 Not Modified` response then ends the play, because the catalog has not changed since the local manifest was saved.
3. Reads the new manifest and the reference manifest as CSV rows with the columns `file`, `hash`, and `size`. There is no header row. Each row is keyed by the file name.
4. Downloads every OSV file that appears in the new manifest and not in the reference. For each file it prints the severity score, ecosystem, package name, fixed version, and published or modified timestamps.
5. Downloads every OSV file that exists in both manifests when the hash has changed, and prints the same details for those modified records.
6. Replaces `<technology>/PULP_MANIFEST.ref` with the manifest just downloaded. The previous reference is kept as a backup (`PULP_MANIFEST.ref.~1~`, and so on).

OSV files are fetched with `force: true`, so a cached `304` cannot hide a record that the manifest says is new or modified.

## Requirements

- Ansible (the `ansible.builtin` modules used here ship with ansible-core).
- The `community.general` collection, which provides `community.general.read_csv`.

```bash
ansible-galaxy collection install community.general
```

Confirm the tasks the play will run:

```bash
ansible-playbook security.lightwell.osv_get --list-tasks
```

## Credentials

`lightwell_user` and `lightwell_token` are not stored in the playbook. The play loads them from the environment on each run:

| Environment variable | Playbook variable | Used as | Description |
| --- | --- | --- | --- |
| `LIGHTWELL_USER` | `lightwell_user` | HTTP basic username | `LIGHTWELL_USER` is the account that can read the Lightwell Pulp repository |
| `LIGHTWELL_TOKEN` | `lightwell_token` | HTTP basic password | `LIGHTWELL_TOKEN` is that account's token. |

Export them in the shell that launches Ansible. Use single quotes so the shell does not alter the token.

```bash
export LIGHTWELL_USER='1234567|lightwell-poc'
export LIGHTWELL_TOKEN='your-token'
```

A missing or empty value fails the first task, before any request is sent:

```text
Set LIGHTWELL_USER and LIGHTWELL_TOKEN in the environment before running this playbook. See README.md.
```

Do not commit the token, and do not put it back into the playbook or into extra vars that get logged.

## First run

The comparison reads `./<technology>/PULP_MANIFEST.ref`. When that file is missing, the play copies `./<technology>/PULP_MANIFEST` to it and continues.
The first comparison uses that copy as the baseline, so it reports no new or modified files. Later runs report files added or changed after this point.

Java, the default:

```bash
export LIGHTWELL_USER='1234567|lightwell-poc'
export LIGHTWELL_TOKEN='your-token'

ansible-playbook security.lightwell.osv_get
```

Python:

```bash
ansible-playbook security.lightwell.osv_get -e lightwell_tech_osv=python
```

If the upstream manifest is still the same file just saved, the following run returns HTTP 304 and exits without printing vulnerabilities.
That is the expected idle result.

To force a full download and comparison against the baseline you just saved:

```bash
ansible-playbook security.lightwell.osv_get -e lightwell_manifest_force=true
ansible-playbook security.lightwell.osv_get -e lightwell_tech_osv=python -e lightwell_manifest_force=true
```

When the two manifests match, the new and modified lists are empty. The play still refreshes `<technology>/PULP_MANIFEST.ref` from `<technology>/PULP_MANIFEST`.

## Sample commands

Check the Java catalog and print new or modified vulnerabilities:

```bash
export LIGHTWELL_USER='1234567|lightwell-poc'
export LIGHTWELL_TOKEN='your-token'

ansible-playbook security.lightwell.osv_get
ansible-playbook security.lightwell.osv_get -e lightwell_tech_osv=java
```

Check the Python catalog. Its manifest is stored in `./python/`, separate from the Java files in `./java/`.

```bash
ansible-playbook security.lightwell.osv_get -e lightwell_tech_osv=python
```

Show the HTTP result, both manifest dictionaries, and the file lists. Those `debug` tasks are silent unless verbosity is at least 1.
The vulnerability messages themselves always print.

```bash
ansible-playbook security.lightwell.osv_get -v
```

Ignore a local copy and download the manifest again. Leave this at `false` for routine runs so an unchanged catalog stops on HTTP 304.

```bash
ansible-playbook security.lightwell.osv_get -e lightwell_tech_osv=java -e lightwell_manifest_force=true
ansible-playbook security.lightwell.osv_get -e lightwell_tech_osv=python -e lightwell_manifest_force=true
```

Point the play at another Pulp base URL. The remediated path is appended as `/osv/<technology>/<type>`, unless you also override `lightwell_osv_url`.

```bash
ansible-playbook security.lightwell.osv_get \
  -e lightwell_tech_osv=python -e lightwell_type_osv=predisclosure \
  -e lightwell_pulp_base_url='https://packages.redhat.com/api/pulp-content/lightwell'
```

## What a finding looks like

A file that was not in the reference:

```text
New OSV file 'GHSA-example.json' contains a vulnerability of severity '7.5'
concerning the Maven package 'org.example:library', fixed in version '1.2.3'.
Published: 2026-01-15T00:00:00Z.
Someone should do something about it...
```

A file whose hash changed:

```text
Modified OSV file 'GHSA-example.json' contains a vulnerability of severity '7.5'
concerning the Maven package 'org.example:library', fixed in version '1.2.4'.
Published: 2026-01-15T00:00:00Z, Modified: 2026-02-01T00:00:00Z.
Someone should look into it...
```

The message is built from `severity[0].score`, `affected[0].package.ecosystem`, `affected[0].package.name`, `affected[0].ranges[0].events[1].fixed`, `published`,
and `modified`. A missing score is printed as `unknown`.

## Playbook variables

| Variable | Default | Role |
| --- | --- | --- |
| `lightwell_user` | `LIGHTWELL_USER` | HTTP username. Set only through the environment. |
| `lightwell_token` | `LIGHTWELL_TOKEN` | HTTP password. Set only through the environment. |
| `lightwell_pulp_base_url` | `https://packages.redhat.com/api/pulp-content/lightwell` | Pulp content root. |
| `lightwell_tech_osv` | `java` | Technology to track. `java` or `python` or `javascript`. |
| `lightwell_type_osv` | `remediated` | Type of fix. `remediated` or `predisclosure`. |
| `lightwell_osv_url` | `{{ lightwell_pulp_base_url }}/osv/{{ lightwell_tech_osv }}/{{ lightwell_type_osv }}` | Directory of OSV files for the selected technology. |
| `lightwell_osv_manifest_url` | `{{ lightwell_osv_url }}/PULP_MANIFEST` | Manifest URL. |
| `lightwell_base_osv_dir` | Current directory as per environment variable `PWD` (Linux) or `CD` (Windows) | Local directory where to create our directory structure. |
| `lightwell_osv_dir` | `{{ lightwell_base_osv_dir }}/{{ lightwell_tech_osv }}/{{ lightwell_type_osv }}` | Local directory for this technology's manifest files. |
| `lightwell_osv_manifest_file` | `{{ lightwell_osv_dir }}/PULP_MANIFEST` | Local copy of the manifest. |
| `lightwell_osv_manifest_ref` | `{{ lightwell_osv_dir }}/PULP_MANIFEST.ref` | Previous manifest used as the baseline. |
| `lightwell_manifest_force` | `false` | When `true`, download the manifest even if the server would return 304. |

Override the technology, URL, path, or `lightwell_manifest_force` values with `-e`. Leave the two credential variables to the environment.

## Files left in the work directory

Each technology keeps its own manifest. Java and Python baselines stay independent.

| File | Meaning |
| --- | --- |
| `<technology>/<type>/PULP_MANIFEST` | Latest manifest downloaded from Pulp. |
| `<technology>/<type>/PULP_MANIFEST.ref` | Baseline updated at the end of a run that got HTTP 200. |
| `<technology>/<type>/PULP_MANIFEST.ref.~N~` | Backup of the previous baseline, created by the final `copy` task. |

A run that ends on HTTP 304 does not replace the reference (unless it is forced).
