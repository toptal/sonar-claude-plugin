# Sonar Claude Code plugin

This repository is a public mirror of the Claude Code plugin that ships
with [Sonar](https://sonar.toptal.com). It exists so anyone can install
the plugin without needing access to the private Sonar source
repository.

## Install

```
/plugin marketplace add toptal/sonar-claude-plugin
/plugin install sonar@sonar
```

Or via the branded URL:

```
/plugin marketplace add https://sonar.toptal.com/claude/marketplace.json
```

You will need a Sonar API key. See the plugin's own README under
[`sonar/`](./sonar/) for the full setup, authentication, and usage
guide.

## Do not edit files here directly

The `sonar/` folder is auto-published from
[`toptal/sonar`](https://github.com/toptal/sonar): on every merge to
`master` that touches `claude-plugin/**` or the mirror-payload/workflow
files, and via manual dispatch of the "Publish Claude plugin to public
mirror" workflow. Any manual change here will be overwritten on the
next publish. Send changes as pull requests against the private sonar
repo instead.
