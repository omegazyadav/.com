## Local Deployment

### Pull the latest themes

```bash
git submodule update --init --recursive
git submodule update --remote --merge

```

### Start the Hugo server locally

```bash
hugo server -D --themesDir ./themes
```
