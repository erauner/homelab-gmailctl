# Gmail Filter Configuration

Version-controlled Gmail filters using [gmailctl](https://github.com/mbrt/gmailctl).

## Setup

### 1. Install gmailctl

```bash
brew install gmailctl
# or
go install github.com/mbrt/gmailctl/cmd/gmailctl@latest
```

### 2. Authenticate

```bash
gmailctl init
```

This will:
- Open a browser for Google OAuth
- Create `~/.gmailctl/` with credentials

### 3. Link config

```bash
# Option A: Symlink (recommended)
ln -sf $(pwd)/config.yaml ~/.gmailctl/config.yaml

# Option B: Copy
cp config.yaml ~/.gmailctl/config.yaml
```

## Usage

### Preview changes
```bash
gmailctl diff
```

### Apply filters
```bash
gmailctl apply
```

### Export current state (from Gmail)
```bash
gmailctl export > exported.yaml
```

### Validate config
```bash
gmailctl test
```

## Filter Categories

| Category | Purpose |
|----------|---------|
| **GitHub** | CI notifications, PRs, issues - archived + marked read |
| **LinkedIn** | All LinkedIn spam - archived |
| **Jobs** | Job alerts from various sites - archived for review |
| **Newsletters** | Substack, Beehiiv, etc - archived |
| **Receipts** | Financial/order confirmations - archived |
| **VIP** | Direct human emails - marked important |

## Maintenance

When adding new filters:
1. Edit `config.yaml`
2. Run `gmailctl diff` to preview
3. Run `gmailctl apply` to deploy
4. Commit changes

## Raw Exports

- `filters.json` - Raw Gmail API filter export
- `labels.json` - Raw Gmail API label export

These are kept for reference/migration purposes.

## Notes

- Filters are additive - multiple filters can match the same email
- `archive: true` removes from inbox but keeps in All Mail
- `delete: true` moves to Trash
- Categories: `personal`, `social`, `updates`, `forums`, `promotions`
