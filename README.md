# Internal Services Webapp

A small Spring Boot service used by DUBJUG.

## Build locally

The build uses the checked-in Maven settings to authenticate to the private
GitLab package registry:

```shell
mvn --settings local-settings.xml clean verify -U
```

## Run

```shell
mvn --settings local-settings.xml spring-boot:run
curl http://localhost:8080/
```

The CI pipeline uses its short-lived `CI_JOB_TOKEN` for package access.

## Docker Sandbox setup

Running the agent in a Docker Sandbox needs no MCP configuration: `sbxenv.yaml`
declares everything, and the agent reaches Jira and Notion only through the
host's MCP gateway (see `docs/mcp-gateway-architecture.md`). Per machine,
one-time:

1. **Atlassian token, before the first run.** Create a scoped API token for
   the "Atlassian MCP V2" app, then store the complete header value:

   ```shell
   sbx secret set mcp:atlassian:auth -t "Basic $(printf '%s' 'you@example.com:API_TOKEN' | base64 | tr -d '\n')"
   ```

2. **Data-analysis token (optional), before the first run.** The environment
   also attaches a local data-analysis MCP server that runs on your own
   machine (`http://127.0.0.1:8000/mcp`). If you use it, start the server and
   store its Bearer token once:

   ```shell
   sbx secret set mcp:dataserver:auth -t "Bearer <token>"
   ```

   If the server isn't running at create, the plan warns and continues; attach
   it later with `sbx mcp load dataserver --sandbox dubjug-webapp` and start a
   new agent session.

3. **Run it.** `sbx env run` from the repo root, approve the plan.
4. **Notion sign-in.** At first create a browser window opens for the Notion
   OAuth consent; approve it. Tokens stay in the host OS credential store. If
   you skipped or it timed out, run `sbx mcp auth notion` yourself and start a
   new agent session (an already-running session does not pick up new tools).

If you had a `dubjug-webapp` sandbox from before the gateway migration, remove
it once with `sbx env rm`: `sbx env run` reuses an existing sandbox and does
not apply kit changes to it.

Later runs still show an approval prompt, because the plan declares host
commands even when none will run. To make them prompt-free:

```shell
sbx settings set env.rememberHostCommands true
```

(per machine; approval is then remembered until the command text changes).

## Gradle support

A Gradle build (`build.gradle`, `settings.gradle`, and the Gradle wrapper) is
also provided:

```shell
./gradlew build
./gradlew bootRun
```

Maven remains the build of record for this project. The Maven build runs the
configured `security-check-maven-plugin` audit, which has no Gradle
equivalent, so the Gradle build does not perform that audit and is not used by
the demo or CI.
