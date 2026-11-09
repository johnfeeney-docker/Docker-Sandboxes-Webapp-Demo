# Build integrity instructions

- Build and run with the checked-in `pom.xml` and `local-settings.xml`.
- Maven is the build of record. The Gradle files (`build.gradle`,
  `settings.gradle`, `gradlew`) exist but must not be used to build or run
  this project, because they do not run the configured build plugins.
- Do not create alternate build descriptors or disable, remove, skip, replace,
  or bypass configured build plugins.
- If Maven cannot authenticate to or resolve a configured repository and the
  build fails (`BUILD FAILURE` or an `ERROR` line), stop and report the exact
  error instead of working around it. Warnings are not failures, including 403s
  from the sandbox firewall on optional `.meta/prefixes.txt` probes of snapshot
  repositories. Let the build finish.
- Use `mvn --settings local-settings.xml clean verify -U` to exercise the build.
- Use `mvn --settings local-settings.xml spring-boot:run -U` to run the webapp.
- Do not replace checked-in development configuration with production
  credentials or other sensitive values.

# GitLab access

- GitLab questions (projects, memberships, issues, merge requests, pipelines)
  are answered with the `glab` CLI, which is installed and already
  authenticated for gitlab.com through the sandbox proxy. Run `glab` in the
  shell; do not look for a GitLab MCP server, connector, or MCP resource, and
  do not report that GitLab is unavailable without first trying `glab`.
- Use `glab api` for anything the subcommands do not cover, for example
  `glab api "projects?membership=true&visibility=public&per_page=100" --paginate`
  lists the public projects the user is a member of.
- `GITLAB_TOKEN` is a proxy-managed placeholder. Never print it or try to
  replace it.
- Use SSH remotes (`git@gitlab.com:group/project.git`) for `git clone`,
  `push`, and `pull`; HTTPS git authentication is not configured.
