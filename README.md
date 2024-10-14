# octotail

Live tail GitHub Action runs on `git push`. It's cursed.

![](https://raw.githubusercontent.com/rarescosma/octotail/a9c662e5f669c22c591d93c32cdeca68e1a05aec/var/demo.gif)

## Motivation

I *really* liked how [Codecrafters][] test runs are mirrored back right in the 
terminal when you `git push`, so I thought: "surely this is something the gh
CLI supports". [It doesn't.](https://github.com/cli/cli/issues/3484)

A couple of hours of messing with HTTPS mitm proxies, websockets, headless
browsers, you-name-it, and __octotail__ was born.

## Wait, what?!

Invoked with a `commit_sha` (and optionally `--workflow-name` 
and/or `--ref-name`), it polls the GitHub API for a matching workflow run. 
When a job associated with the run starts, it instructs a headless 
chromium-based browser to visit the job's page.

The browser's traffic passes through a [mitmproxy][] instance that it uses 
to extract the authenticated WebSocket subscriptions for live tailing.

The WebSocket address and subscribe messages are then passed to the tailing 
workers.

The headless browser tabs are cleaned up immediately after the WebSocket
extraction, so the overhead is minimal. (well, it's still an empty browser)

## Installation

### Pre-install

- python 3.12
- a working chromium-based browser under `/usr/bin/chromium`

Make sure `/usr/bin/chromium` points to a working chromium-based browser.

If unsure, and on Arch:

```shell
paru ungoogled-chromium-bin
```

### Via git and make

```shell
git clone https://github.com/rarescosma/octotail.git
cd octotail
make
sudo make install
```

### Via git - all manual

Clone the repo:

```shell
git clone https://github.com/rarescosma/octotail.git
cd octotail
```

Make a virtual environment, activate it, and install the package.

```shell
python3 -m venv .venv
source .venv/bin/activate
poetry install --no-dev
sudo ln -sf $(pwd)/octotail/main.py /usr/local/bin
```

### __Important:__ post-install

Run `mitmproxy` once and install its root certificate:

```shell
mitmproxy
^C

sudo trust anchor ~/.mitmproxy/mitmproxy-ca-cert.cer
```

## Usage

```
# octotail --help
                                                                                                    
 Usage: octotail [OPTIONS] COMMIT_SHA                                                               
                                                                                                    
 Look for an active workflow run for the given <COMMIT_SHA> (and optionally --workflow-name and/or  
 --ref-name) and attempt to tail its logs.                                                          
 NOTE: the <COMMIT_SHA> has to be of the full 40 characters length.                                 
                                                                                                    
╭─ Arguments ──────────────────────────────────────────────────────────────────────────────────────╮
│ *    commit_sha      TEXT  Full commit SHA that triggered the workflow. [default: None]          │
│                            [required]                                                            │
╰──────────────────────────────────────────────────────────────────────────────────────────────────╯
╭─ Options ────────────────────────────────────────────────────────────────────────────────────────╮
│ *  --gh-pat                         TEXT     GitHub personal access token. [env var: _GH_PAT]    │
│                                              [required]                                          │
│ *  --gh-user                        TEXT     GitHub username. (for web auth) [env var: _GH_USER] │
│                                              [required]                                          │
│ *  --gh-pass                        TEXT     GitHub password. (for web auth) [env var: _GH_PASS] │
│                                              [required]                                          │
│    --gh-otp                         TEXT     GitHub OTP. (for web auth) [env var: _GH_OTP]       │
│                                              [default: None]                                     │
│    --workflow  -w                   TEXT     Look for workflows with this particular name.       │
│                                              [default: None]                                     │
│    --ref-name  -r                   TEXT     Look for workflows triggered by this ref.           │
│                                              Example: 'refs/heads/main'                          │
│                                              [default: None]                                     │
│    --headless      --no-headless             [env var: _HEADLESS] [default: headless]            │
│    --port                           INTEGER  [env var: _PORT]                                    │
│                                              [default: (random in range 8100-8500)]              │
│    --help                                    Show this message and exit.                         │
╰──────────────────────────────────────────────────────────────────────────────────────────────────╯
```

### Tail after push

A simple use case is tailing a workflow run right after `git push`:

```shell
git push origin main
octotail $(git rev-parse origin/main) -r refs/heads/main
```

Or if you're pushing a tag:

```shell
git push origin v1.0.42
octotail $(git rev-parse v1.0.42^{commit}) -r refs/tags/v1.0.42
```

### As a post-receive hook

A slightly more advanced use case that lets you stream the run outputs on
`git push`, similar to how you get the test runs results when pushing
to [Codecrafters][].

For this to work we'll need control over the remote's output, so we can't use
the GitHub remote directly. Instead, we'll use a bare repository as our "proxy"
remote and set up its post-receive hook to call `octotail`.

```shell
cd your-original-repo
export PROXY_REPO="/wherever/you/want/to/store/the/proxy-repo"

mkdir -p $PROXY_REPO
git clone --mirror "$(git remote get-url origin)" $PROXY_REPO
git remote add proxy $PROXY_REPO
# back to octotail
cd -

cp post-receive.sample $PROXY_REPO/hooks/post-receive
```

Edit `$PROXY_REPO/hooks/post-receive` and change things according to 
your setup:

- set `_GH_USER` to your GitHub username
- set `_GH_PASS_CMD` to a command that outputs the GitHub password, e.g. 
  `_GH_PASS_CMD="pass github.com"`
- _if using 2FA_ - set `_GH_OTP_CMD` to a command that outputs an OTP token 
  for the GitHub 2FA, e.g. `_GH_OTP_CMD="totp github.com"`
- set `_GH_PAT_CMD` to a command that outputs your GitHub personal access token,
  e.g. `_GH_PAT_CMD="pass github_pat"`

NOTE: the hook assumes you're using `zsh`. You can change the shebang to your 
own shell, but you might want to invoke it with the right flags to get an 
interactive, login shell. Useful to get access to custom functions and aliases.

That's it! (phew) - now try pushing some commits to the `proxy` remote and check
if you get the GitHub Actions run logs streaming right back:

```shell
cd your-original-repo
git commit --allow-empty -m 'test octotail'
git push proxy
```

[Codecrafters]: https://codecrafters.io/
[mitmproxy]: https://mitmproxy.org/
