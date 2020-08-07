# countme-totals: DNF user counting data

This repo contains aggregate `countme` data about DNF clients gathered from
https://mirrors.fedoraproject.org/.

For more information about how this data is gathered, see the documentation for
the `mirrors-countme` scripts: https://pagure.io/mirrors-countme/

## What's here?

* `totals.db`: total counts, in a SQLite database
* `totals.csv`: same, in CSV format (with a header line)
* `mirrorsdata-2020.csv`: Counts of unique IP addresses, from
  https://data-analysis.fedoraproject.org/csv-reports/mirrors/mirrorsdata-2020.csv
* `jupyter/`: [Jupyter] notebooks showing how to use this data
  * `jupyter/dnf-countme-pandas-demo.ipynb`:
    how to summarize, slice, and graph data from `totals.csv` in Python, using [pandas]

[Jupyter]: https://jupyter.org/
[pandas]: https://pandas.pydata.org/

## What's being counted?

This data counts all valid `countme` requests (grouped per-week)
since the feature was first enabled in Fedora 32, on 2020-02-11.

It's broken down by:

* The client OS name, version, variant, and arch,
* the "age" of the system (on a scale from 1-4), and
* which repo (and arch) it requested.

The actual requests look (roughly) like this:

    User-Agent: libdnf (Fedora 32; workstation; Linux.x86_64)
    GET /metalink?repo=fedora-modular-32&arch=x86_64&countme=1 HTTP/2.0

We split this up into 7 pieces of info:

    User-Agent: libdnf ({os_name} {os_version}; {os_variant}; Linux.{os_arch})
    GET /metalink?repo={repo_tag}&arch={repo_arch}&countme={sys_age} HTTP/2.0

And so each item in `totals.db` and `totals.csv` is a count of how many times
we saw that particular combination of:

    (os_name, os_version, os_variant, os_arch, sys_age, repo_tag, repo_arch)

in a given week.


### `totals.csv`

The CSV version of this data contains a header line, and it looks like this:

```
week_start,week_end,hits,os_name,os_version,os_variant,os_arch,sys_age,repo_tag,repo_arch
2020-02-10,2020-02-16,2,Fedora,31,generic,x86_64,3,fedora-modular-31,x86_64
2020-02-10,2020-02-16,2,Fedora,31,generic,x86_64,3,updates-released-f31,x86_64
2020-02-10,2020-02-16,1,Fedora,31,workstation,x86_64,3,updates-released-modular-f31,x86_64
2020-02-10,2020-02-16,2,Fedora,31,workstation,x86_64,3,fedora-31,x86_64
...
2020-07-13,2020-07-19,2,Fedora,32,workstation,x86_64,1,rawhide-modular,x86_64
2020-07-13,2020-07-19,1,Fedora,32,,x86_64,1,fedora-32,x86_64
2020-07-13,2020-07-19,1,Fedora,32,silverblue,x86_64,1,fedora-rawhide,x86_64
2020-07-13,2020-07-19,1,Fedora,32,workstation,x86_64,3,updates-released-f33,x86_64
2020-07-13,2020-07-19,1,Fedora,32,workstation,x86_64,3,fedora-33,x86_64
```

The columns are:

1. `week_start`
: ISO-formatted date for the first day (Monday) of the week

2. `week_end`
: ISO-formatted date for the last day (Sunday) of the week

3. `hits`
: number of requests during this week

4. `os_name`
: `NAME` from `/etc/os-release`

5. `os_version`
: `VERSION_ID` from `/etc/os-release`

6. `os_variant`
: `VARIANT_ID` from `/etc/os-release`

7. `os_arch`
: dnf `$basearch`

8. `sys_age`
: System age, 1-4

9. `repo_tag`
: the requested URL's `repo=` value

10. `repo_arch`
: the requested URL's `arch=` value.


### `totals.db`

The SQLite version of this data is in a table named `countme_totals` and looks like this:

| `hits` | `weeknum` | `os_name` | `os_version` | `os_variant` | `os_arch` | `sys_age` | `repo_arch` | `repo_tag`                   |
| ------ | --------- | --------- | ------------ | ------------ | --------- | --------- | ----------- | ---------------------------- |
| 71482  | 2636      | Fedora    | 32           | workstation  | x86\_64   | 3         | x86\_64     | updates-released-f32         |
| 70546  | 2636      | Fedora    | 32           | workstation  | x86\_64   | 3         | x86\_64     | updates-released-modular-f32 |
| 67035  | 2636      | Fedora    | 32           | workstation  | x86\_64   | 3         | x86\_64     | fedora-modular-32            |
| 67001  | 2636      | Fedora    | 32           | workstation  | x86\_64   | 3         | x86\_64     | fedora-32                    |
| 28674  | 2636      | Fedora    | 32           | workstation  | x86\_64   | 1         | x86\_64     | updates-released-f32         |
| ...    | ...       | ...       | ...          | ...          | ...       | ...       | ...         | ...                          |

Instead of `week_start` and `week_end`, we use `weeknum`, where weeks start on
Monday at 00:00:00 UTC, and week 0 started on the first Monday of the UNIX
Epoch (Mon Jan 5 1970).

To convert a `weeknum` to a POSIX timestamp:

    ((weeknum*7)+4)*86400

You can thus convert `weeknum` to a SQLite `date` (or `datetime`):

    date(((weeknum*7)+4)*86400,'unixepoch')

But it's actually easier to use the Julian day number:

    date(julianday('1970-01-05')+weeknum*7)
    date(2440591.5+weeknum*7)


### Counting hits vs. hosts

One important point about this data: it counts _hits_, not _hosts_. We can
approximate the number of hosts we saw by grouping the counts by all the host
identifiers (`os_name`, `os_version`, `os_variant`, `os_arch`, `sys_age`) -
which gives us hits for that host type across all repos - and taking the
`max()` of that set. Like so:

```
select date(2440591.5+weeknum*7) as week_start,
       os_name, os_version, os_variant, os_arch, sys_age,
       sum(hits) as totalhits, max(hits) as hosts
from countme_totals
where os_name == 'Fedora' and os_version=='31' and os_arch=='x86_64' and weeknum==2636
group by os_name, os_version, os_variant, os_arch, sys_age;
```
