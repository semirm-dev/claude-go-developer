# claude-go-developer

A Claude Code subagent (`go-developer`) and the Go skills it uses. They cover my Go guidelines plus playbooks for concurrency, memory, Gin servers, SPA auth, gRPC and Kubernetes operators.

```
agents/go-developer.md      the subagent; preloads go-guidelines, loads topic skills on demand
skills/go-guidelines/       naming, structs, interfaces, errors, dependencies, layout
skills/go-concurrency/      goroutine lifecycle, channels, worker pool, pipeline, errgroup
skills/go-memory/           pprof, GOGC/GOMEMLIMIT, boxing, slices/maps, sync.Pool
skills/go-gin-server/       timeouts, shutdown, middleware, handlers, handler tests
skills/go-spa-auth/         access/refresh tokens, CSRF, OAuth + PKCE
skills/go-grpc/             buf, proto evolution, servers, streaming
skills/go-k8s-operators/    CRDs, reconcile, status, finalizers, testing
```

Each skill has a `SKILL.md` and runnable reference code in `references/`.

## Install

### Option 1: symlink (recommended)

Edits you make in `~/.claude` land in this repo, so you can commit them.

```bash
git clone git@github.com:semirm-dev/claude-go-developer.git ~/workspace/claude-go-developer
cd ~/workspace/claude-go-developer
mkdir -p ~/.claude/agents ~/.claude/skills
ln -s "$PWD/agents/go-developer.md" ~/.claude/agents/
for s in skills/go-*; do ln -s "$PWD/$s" ~/.claude/skills/; done
```

### Option 2: copy

```bash
git clone git@github.com:semirm-dev/claude-go-developer.git
cd claude-go-developer
mkdir -p ~/.claude/agents ~/.claude/skills
cp agents/go-developer.md ~/.claude/agents/
cp -R skills/go-* ~/.claude/skills/
```

To pull in updates later, run `git pull` and copy the files again.

### Per project only

To limit them to one project, put them in that project's `.claude/` folder instead of `~/.claude/`:

```bash
mkdir -p .claude/agents .claude/skills
cp /path/to/claude-go-developer/agents/go-developer.md .claude/agents/
cp -R /path/to/claude-go-developer/skills/go-* .claude/skills/
```

## Check it works

Start a new Claude Code session, then:

- run `/agents`: `go-developer` should be listed
- the `go-*` skills should appear in the session's skill list (or ask "which Go skills do you have?")

## Notes

- The gRPC (`go-grpc`) and Kubernetes (`go-k8s-operators`) references need generated code before they compile: run `buf generate` in `f1-protobuf-and-buf/`, and `controller-gen object paths=./...` for the operator examples.
- The other references pass `go vet` and `go test -race`.
