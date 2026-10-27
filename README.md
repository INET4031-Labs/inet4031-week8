# inet4031-week8 (Week 8 Lab Reference Repo)

Week 8 of INET 4031: security hardening and shifting left. Students harden their web app and database through the delivery loop they built in Week 7, add a security scan to the deploy gate so it runs on pull requests, turn on a GitHub ruleset that refuses to merge a failing pull request, and change the deploy script so it waits for a person before it deletes anything. They then attack their own loop twice, once with a pull request that loosens a setting and once with a valid change that would delete their database volume, and watch two different layers stop them.

## Where the lab is

The step-by-step instructions are in this repo's [wiki](https://github.com/INET4031-Labs/inet4031-week8/wiki). Students follow the wiki, not this README.

## Nothing to clone this week

As in Weeks 5 to 7, there are no starter files. Students create or edit every file themselves from the wiki, inside the personal repo they have used all semester. They do not clone this repo.

## What students end up with

In their own repo, committed through pull requests:

```
.github/
  workflows/
    deploy-gate.yml     (replaced: now runs on pull requests, with a security scan job)
week-5/
  web.tf                (rewritten: non-root user, no privilege escalation, limits)
  db.tf                 (edited: CPU and memory limits)
  network.tf            (new: only web pods may reach the database)
week-7/
  deploy.sh             (edited: saves the plan and stops on any deletion)
week-8/
  README.md
  ruleset.json
```

Outside the repo:

```
GitHub          a ruleset named protect-master (created from week-8/ruleset.json)
the VM          tighter permissions on the state file, the deploy log and the home folder
```

No new passwords or keys are created this week, and `.gitignore` does not change. The Week 5 rules still keep these on the VM and out of Git:

```
week-5/terraform.tfvars
week-5/terraform.tfstate
week-5/terraform.tfstate.backup
week-5/.terraform/
```

## Prerequisites

- Week 7 finished, with the deploy gate, `week-7/deploy.sh` and the cron job all working. The last lines of `~/deploy.log` should say `up to date`.
- The Week 5 and Week 6 stack running: `web` and `db` `Running`, and the three Prometheus pods `Running` in `monitoring`.
- The personal repo from Week 3, still public, with a clean working tree on `master`.
- The GitHub CLI signed in as the owner of that repo. Creating a ruleset needs admin rights on the repo, which the owner has.
- k3s with its default network policy controller. The lab's NetworkPolicy does nothing if k3s was installed with `--disable-network-policy`.
- The Ubuntu VM from Week 1, with internet access. The test pods pull `postgres:16`, which k3s already has from Week 5 if nothing has cleaned it up.

## What the lab covers

- What hardening and shift left mean, and where each applies in the delivery loop
- Taking an inventory of what a VM exposes: open ports, file permissions, and the user a container runs as
- Why OpenTofu's state file is readable by every user on the VM by default, and fixing it
- A security context for the web container (non-root user, no privilege escalation, no capabilities), resource limits for both Deployments, and a NetworkPolicy that lets only `web` pods reach the database
- A policy-as-code scan: Checkov, run in the gate on every pull request, checking six rules
- A GitHub ruleset that requires a pull request and three passing checks, with no bypass for anyone
- Changing the deploy script so it saves the plan, counts deletions and replacements, and stops with `BLOCKED` until a person approves
- Attack 1, a pull request that sets `allow_privilege_escalation = true`, which the scan fails and the ruleset refuses to merge
- Attack 2, a pull request that renames the database volume claim, which passes every check and is stopped only by the deploy script
- Rolling back the rename, and showing the database volume was never replaced

## Things to know before running it

- Written against OpenTofu 1.12.5 in the gate, version 3 of the Kubernetes provider (a dry run used 3.3.0), and Checkov 3.3.26. The gate installs Checkov from PyPI at that one exact version, and it uses the same pinned commits for `actions/checkout` and `opentofu/setup-opentofu` as Weeks 6 and 7.
- Checkov is installed with `pip` and not through a ready-made GitHub Action on purpose. In March 2026 attackers rewrote the version tags of a popular scanner's Action so that pipelines using them ran credential-stealing code. The wiki uses this to explain why the earlier weeks pinned actions to commits.
- The scan checks six rules: CPU and memory requests and limits (`CKV_K8S_10` to `_13`), no privileged containers (`_16`) and no privilege escalation (`_20`). This scanner has no rule for a non-root user on OpenTofu files, so the wiki proves that change with `kubectl exec deploy/web -- id` instead.
- The gate has no `paths` filter on purpose. The ruleset requires the gate's three jobs, and a required job that a filter skips never reports, which would block the merge forever. Last week's `opentofu-checks.yml` has a filter, so it is not on the required list.
- The ruleset matches checks by job name, not by workflow name. `gh pr checks` prints them as `Deploy gate/<job name>`, and the ruleset uses only the part after the slash.
- `bypass_actors` is empty, so even the repo owner cannot merge a failing pull request. The wiki shows how to delete the ruleset if a check name is wrong.
- `web.tf` is replaced from the wiki, and the replica count is captured first and restored. Students can start at one replica or two, and the dry run started at two.
- `db.tf` is edited by hand: a short `resources` block pasted above `readiness_probe {`. If it is pasted without indentation, `tofu fmt` repairs it. The check is `grep -c 'limits = {'`, which should print `1`. Do not check for `resources {`, because the storage claim in the same file already has one.
- The network test uses two pods that only sleep, then runs `pg_isready` from inside each. `probe` is blocked, and `probe-web`, which carries the `app=web` label, gets through. The policy matches labels, not identity. The earlier `kubectl run --rm -i` version lost its output when the pod finished quickly, and a new pod can be blocked for a few seconds by mistake before the policy reaches it, which is what the `sleep 20` is for.
- `gh pr merge` with no name merges the pull request for the branch you are on. After a `git checkout master` it finds nothing and prints `no pull requests found for branch "master"`. Every merge in the wiki names its pull request.
- The new `deploy.sh` is already on disk when the hardening deploy runs, because `gh pr merge` updates the local `master` the moment it merges. The first deploy therefore prints `step 1 of 3: tofu plan`.
- The approval step runs after the working folder moves forward, because the plan needs the new files. A blocked commit stays in the working copy, and `BLOCKED` repeats every two minutes, until a newer commit arrives that makes the plan clean. The log prints the whole plan before `BLOCKED`, which looks long. Students must not run `tofu apply` while the rename is blocked, because it would delete the database volume.
- The step counts `will be destroyed` and `must be replaced` in the plan's text. That is easy to read and good enough here, and a resource whose own values contain those words would cause a false block.
- OpenTofu creates `terraform.tfstate` with the default umask, which makes it world-readable, and a backup file created later is also world-readable. The wiki fixes the existing files with `chmod`, and the new script sets `umask 077` so the files it creates are private.
- In a dry run, the three gate jobs took between 6 and 41 seconds (the scan is the slow one), and the deploy itself took about 40 seconds from `new commit` to `deployed`. The `db` pod restarts during the hardening deploy because its Deployment uses `Recreate`, and the data stays on the volume.
- The hardening plan came out as `1 to add, 2 to change, 0 to destroy`, and the rename as `1 to add, 1 to change, 1 to destroy`. Both match the wiki.
- Nothing backs up the database, and the wiki says so. Week 9 starts from that gap.

## Common failure points

- **The test pod says `accepting connections` after the deploy.** The hardening pull request has not merged, or the deploy has not run. Check `kubectl get networkpolicy` and `tail ~/deploy.log`. The wiki checks both before the network test.
- **`no pull requests found for branch "master"`.** The merge command was run without a pull request name, from `master`. Use `gh pr merge <branch> --merge --delete-branch`.
- **The log stays `up to date` after Attack 2.** The `rename-volume` pull request never merged. `gh pr list --state merged --limit 1 --json headRefName,state` shows the newest merged pull request.
- **The `security scan` job fails on the hardening pull request.** Usually a paste error in `web.tf` or `db.tf`. `gh run view --log-failed` shows a line starting with `Check:` that names the rule and file.
- **The gate fails at `Initialize`.** The lock file is missing a platform. Run `tofu providers lock -platform=linux_amd64 -platform=linux_arm64` in `week-5/` and commit it, as in Week 6.
- **`gh pr merge` succeeds on the Attack 1 pull request.** The ruleset is not enforcing. Check that it exists with `gh api repos/{owner}/{repo}/rulesets`, that its status is `active`, and that its three check names match the job names.
- **Nothing can merge after the ruleset is created.** A check name does not match. Delete the ruleset with `gh api --method DELETE repos/{owner}/{repo}/rulesets/<id>` and recreate it.
- **`BLOCKED` repeats in the log.** Expected during Attack 2 and until the rollback merges. If it appears at any other time, read the plan with `cd week-5 && tofu plan` before doing anything.
- **`SKIPPED: this folder has commits that are not on origin/master`.** A commit landed on the local `master` instead of on a branch, and the ruleset kept it from reaching GitHub. `git log --oneline origin/master..master` shows it. After saving it on a backup branch, `git reset --hard origin/master` puts the folder back in line.
- **One `SKIPPED: this folder is on '<branch>'` line right after a merge.** Cron ran a few seconds before the student switched back to `master`. The next run carries on.
- **`kubectl get pvc` shows `db-data` only minutes old after Attack 2.** The deploy script was still the Week 7 version, which applies everything it is given, and the rename deleted the volume. `grep -c 'destroys=' week-7/deploy.sh` should print `1` before Attack 2, and the wiki asks for it. The same sign appears in the log as `step 1 of 2: tofu apply` and not `step 1 of 3: tofu plan`.
- **`web` goes into `CrashLoopBackOff` after the hardening deploy.** `kubectl logs deploy/web` shows why. The old pods keep serving while the new one fails, and the fix is a pull request that reverts the merge.
- **`grep -c 'limits = {' db.tf` prints `0` or `2`.** The block was not pasted, or was pasted twice.

## Further reading

- Kubernetes security contexts: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
- Kubernetes NetworkPolicy: https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Checkov: https://www.checkov.io/
- GitHub rulesets: https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets
- GitHub Actions: https://docs.github.com/actions
- OpenTofu: https://opentofu.org/docs/
