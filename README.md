# Pihole Adlist Tool

This script tries to provide you with a bunch of information that enables you to decide which adlists you need based on your browsing behavior. It does that by matching your browsing history (FTL's querylog) with your current adlist configuration (gravity database) generating a list of domains that you have visited in the past and which would have been blocked if your current adlist configuration would have been in place back then.
In a second step the scripts takes this list and attributes each domain to the adlists it is on (similar to what `pihole -q` does).
The final output is a table of all your adlists with the corresponding number of covered domains (domains that you have visited and that would have been blocked if only this particular adlist would have been used).

---

## The script outputs

- the number of adlists (and how many are enabled)
- the number of unique domains in your gravity.db
- the number of blocked domains as reported by Pi-hole ('blocking status == blocked by gravity' or blocking status == blocked by gravity+blocked during CNAME inspection) and how often those domains have been blocked ('hits')
- the number of covered domains and how often those would have been blocked ('hits')
- special case: domains on your (personal) blacklist which are also on an adlist and have been visited in the past, including hits (run 'pihole -q' to see on which adlist those domains appear)
- optional: top blocked domains and number of hits if your current adlist configuration would have been used
- adlist table
    id, status, total domains on adlist, covered domains, hits, unique covered domains, address
- the sum of unique covered domains
- optional: list of unique covered domains with adlist_id, address
- optional: analyse regex blacklist (will be disabled when running Pi-hole in Docker Container!)

As domains usually appear on more then one adlist I introduce the concept of ***unique covered domains***. Those are domains that have been visited, would have been blocked and appear on just one adlist. This might help you to value your adlists not just by how many domains are covered but also what would happen if you disable this adlist.

---

## Limits

- ~~Disabled blocklist won't be analyzed as gravity is not including domains from deactivated adlists. You can enable all adlists from within the script.~~
The script will warn you, if there is a mismatch between the enabled adlists and data found in the gravity database. Users have the choice to run gravity to clear the mismatch or proceed anyway. In this case the tool will analyze all available data, but results must be interpreted with caution. (see [8dab71](https://github.com/yubiuser/pihole_adlist_tool/commit/8dab71836c1b2407c9626b17fd592399a7ef0b58))

- Black/Whitelisted domains (~~including regex~~ see [PR #19](https://github.com/yubiuser/pihole_adlist_tool/pull/19) are not considered when calculating the number of covered domains (and hits)
  - Whitelisted domains reduce the number of blocked domains as reported by Pi-hole compared to the calculated numbers
  - Blacklisted domains increase the number of blocked domains as reported by Pi-hole compared to the calculated numbers

- ~~This tool can not deal with domains that have been blocked due to CNAME inspection because Pi-hole doesn't store the actual blocked domain but the CNAME and a corresponding status ("Blocked during deep CNAME inspection"). This CNAME domain will not match a domain from an adlist - if it would it would have been blocked directly.~~ (see [PR #3](https://github.com/yubiuser/pihole_adlist_tool/pull/3))

- Other differences between the number of domains/hits as reported by Pi-hole and calculated numbers are due to change in adlist configuration over time

- For the limits of the regex analysis see the [notes of PR #19](https://github.com/yubiuser/pihole_adlist_tool/pull/19)

---

## Caveat

- Depending on the number of enabled adlists and the number of visited domains in the selected time period the calculation might take some time - please be patient.
On my [NanoPi NeoPlus2](http://wiki.friendlyarm.com/wiki/index.php/NanoPi_NEO_Plus2)  (ARM, Quad-core Cortex A53)  it takes ~17-18sec to analyse 2.3 million queries from pihole-ftl.db and 347603 domains in gravity.db

- Analysis of regex blacklist can take minutes easily!

- While lists that have attracted no or only very few hits in the analysis are prime candidates for removal, you should also consider the type of blocklist before you ultimately decide do remove a list, e.g. you may want to keep malware or telemetry focused blocklists nonetheless.

---

## Requirements

- Pi-hole FTL v5.5 (see [PR #13](https://github.com/yubiuser/pihole_adlist_tool/pull/13))
- **Notes on Docker**:  I don't run Pi-hole on docker myself and have only limited ability to test the script. Expect things to break anytime.

---

## Usage

```bash
pihole_adlist_tool [options]

Options:
  -d [Num]                        Consider the last [Num] days (Default: 30). Enter 0 for all-time analysis.

  -t [Num]                        Show top blocked domains. [Num] defines the number to show.

  -s [total/covered/hits/unique]  Set sorting order to total (total domains) covered (domains covered), hits (hits covered) or unique (covered unique domains) DESC. (Default sorting: id ASC).

  -u                              Show covered unique domains.

  -a                              Run in 'automatic mode'. No user input is required at all, assuming default choice would be to leave everything untouched.

  -r                              Analyse RegEx as well. Depending on the amount of domains and RegEx this might take a while. Please note: Can only be used, if Pi-hole is NOT running in a Docker Container!

  -v                              Display pihole_adlist_tool's version.

  -h                              Show this help dialog.
```

---

## Background

As adlist configuration might have changed over time (add/removed adlists, enabled/disabled adlists) this script doesn't rely on Pi-holes blocking status for the analysis but rather determine if queries from the long-term database had been blocked with the current adlist configuration. Relying on the blocking status could lead to wrong assumptions about the  coverage of adlist with your current adlist configuration: some domains might have been blocked in the past but wouldn't be blocked now (removed adlist) and some might be blocked now but haven't in the past (added adlist). If the adlist configuration hasn't changed over time, there should be no huge difference between this approach and using Pi-hole's blocking status.

The deeper reason for re-analyzing the queries is that this tool should help you to make predictions for the future: assuming your online behavior is rather stable over time and you analyse a long enough dataset from the past, this tool will tell you which adlist might be worth keeping (because it contains a lot of covered domains) and which you could safely remove (no covered domains and/or covered domains but no unique covered domains).

---

## Support, Contribute & Todo

I'm not a developer. This script is mostly done by copy-pasting snippets I found online. I know there is no proper error and exception handling. If you are willing to improve the script feel free to submit pull requests. Things on my todo list:

- Further improve speed of the database handling. The slow steps are
  - Select all domains from pihole-ftl.db that are also found in gravity.db
  - Get the total number of blocked domains from pihole-ftl.db
  - Get the total number of hits from pihole-ftl.db
  - ~~Update adlist with the total number of domains from gravity.db for each adlist~~ (see  [e0af664](https://github.com/yubiuser/pihole_adlist_tool/commit/e0af6642487515a28c4d1c7eb91f19def634ddce))
- Format SQL output with awk
