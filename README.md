# websh-mempool

Independent public drafts and work in progress for [websh](https://github.com/0xwonj/websh),
mounted at `/mempool`. This source is explicitly unsigned; a root declaration identifies
the source without granting its bodies owner-signature status.

Edit category Markdown files, leave the Git index unstaged, then publish from this
checkout with an installed current `websh-cli`:

```bash
websh-cli mempool publish .
```

The command validates drafts, generates `manifest.json`, creates an immutable snapshot
commit and a `current.json` pointer commit, and pushes once. It does not sign, rebuild
the app, update root content, or upload to IPFS. A failed push can be retried with the
same command without creating another unchanged snapshot.
Push initial setup and later repository-maintenance commits separately with ordinary
Git. Only a prepared snapshot/pointer pair may remain ahead of origin; set newer edits
aside until that pair is published. If origin advanced, reconcile authored inputs onto
the fetched branch and publish again. Do not rebase the prepared pair or force push
published history.

For generation without publication:

```bash
websh-cli mempool sync .
```

Draft frontmatter accepts `title`, `category`, `status`, `priority`, `modified`, and
`tags`; unknown fields fail. Status defaults to `draft`. Category directories determine
categories. Keep one `.md` file per draft directly inside its category. Do not edit
`manifest.json` or `current.json` by hand, and preserve published Git history.

To promote a draft, import it into the separate root content checkout, review, and publish:

```bash
websh-cli --root ../websh-content mempool import writing/example.md
websh-cli --root ../websh-content publish
```

The import does not remove the draft; remove or update its status explicitly and publish
mempool independently. Nothing unpublished or private belongs in this repository.
