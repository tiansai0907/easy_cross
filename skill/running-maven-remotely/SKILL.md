---
name: running-maven-remotely
description: Use when Maven or mvn commands are needed for Java projects, including compile, test, package, install, dependency resolution, or verification from Linux workspaces mounted from a Windows machine.
---

# Running Maven Remotely

## Overview

Maven must run on the remote Windows computer, not on the local Linux workspace.

Core reason: development happens on Linux against mounted project files, but the working Maven environment and local Maven repository exist only on the Windows machine. Local `mvn` results are not authoritative.

## Hard Rule

If a task needs `mvn`, do not run `mvn` locally.

Convert the current local directory to its Windows path, run the Maven command through the configured SSH alias `win-maven`, and use the remote command result for the next plan.

No exceptions for:
- "quick compile checks"
- "just reproducing a test"
- "only packaging"
- "checking Maven version"
- "saving time"
- "the command is read-only"

## Path Mapping

| Local Linux prefix | Remote Windows prefix |
|---|---|
| `~/workspace/projects` | `D:\gitlab_work` |
| `/home/tian/workspace/projects` | `D:\gitlab_work` |

Mapping examples:

| Local directory | Remote directory |
|---|---|
| `/home/tian/workspace/projects/inquest` | `D:\gitlab_work\inquest` |
| `/home/tian/workspace/projects/inquest/com.tian.ff.inquest.business` | `D:\gitlab_work\inquest\com.tian.ff.inquest.business` |
| `/home/tian/workspace/projects/tdzfmkservernote/com.tian.ll.servernote.noteservice` | `D:\gitlab_work\tdzfmkservernote\com.tian.ll.servernote.noteservice` |

## Workflow

1. Identify the local directory where the Maven command should run.
2. Replace the local prefix with `D:\gitlab_work`.
3. Convert `/` to `\` for the Windows path.
4. Run the Maven command over SSH on the Windows computer using `win-maven`.
5. Treat the remote exit code and remote Maven output as the only valid result.
6. If the remote command fails, analyze that failure and update the plan. Do not retry with local `mvn`.

## Command Template

Use `cmd /c` so `cd /d` works even when the drive changes:

```bash
ssh win-maven 'cmd /c "cd /d D:\gitlab_work\<repo>\<subpath> && mvn <goals-and-options>"'
```

Examples:

```bash
ssh win-maven 'cmd /c "cd /d D:\gitlab_work\inquest\com.tian.ff.inquest.business && mvn clean compile -DskipTests"'
```

```bash
ssh win-maven 'cmd /c "cd /d D:\gitlab_work\tdzfmkservernote\com.tian.ll.servernote.noteservice && mvn test"'
```

Use `win-maven` directly. Do not ask for the SSH target unless `ssh win-maven` itself fails.

## Result Handling

After every remote Maven run, report:
- remote directory
- exact SSH command used
- exit status
- key Maven result lines, such as `BUILD SUCCESS`, `BUILD FAILURE`, failed tests, compilation errors, or dependency resolution errors
- next action based on the remote result

If `ssh win-maven` fails, diagnose SSH/connectivity/authentication separately. Do not treat it as a Maven result, and do not run local Maven as a substitute.

## Rationalizations

| Rationalization | Reality |
|---|---|
| "Local `mvn` is faster for a quick check." | Fast but invalid. The local machine lacks the authoritative Maven environment and repository. |
| "I only need to reproduce one test." | Test reproduction still depends on Maven dependencies, plugins, profiles, and repository state from Windows. |
| "Packaging is read-only, so local is harmless." | Read-only does not make the result valid. The build environment is still wrong. |
| "Remote SSH is inconvenient." | Inconvenience is not a reason to collect false build evidence. |
| "I'll try local first, then remote if it fails." | Local success or failure is not authoritative and can mislead the plan. Start remote. |

## Red Lines

Stop and reroute through SSH if you are about to:
- run a command beginning with `mvn`
- run `./mvnw` locally for these mounted projects
- use local Maven output to claim compile, test, package, install, or dependency status
- retry a failed remote Maven command locally
- skip remote Maven because of urgency

The valid action is always: map path, SSH to Windows, run Maven there, judge the remote result.

## Quick Checklist

Before any Maven command:

- [ ] Does this command invoke `mvn` or Maven wrapper behavior?
- [ ] Is the working directory under `/home/tian/workspace/projects` or `~/workspace/projects`?
- [ ] Did I map it to `D:\gitlab_work\...`?
- [ ] Am I using `ssh win-maven 'cmd /c "cd /d ... && mvn ..."'`?
- [ ] Will my next plan be based only on the remote result?

If any answer is "no", do not run the command yet.

## Common Mistakes

- Running `mvn -v` locally to check Maven availability. Check it remotely with SSH.
- Running from the repository root locally because it is "just a reactor check." Reactor checks must be remote too.
- Using Linux path syntax inside the remote command. The remote `cd` path must be the mapped Windows path.
- Ignoring a non-zero SSH exit status. A failed SSH command means Maven did not run successfully.
