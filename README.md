[![CI](https://github.com/aradzie/keybr.com/actions/workflows/ci.yml/badge.svg)](https://github.com/aradzie/keybr.com/actions/workflows/ci.yml)

# [keybr.com](https://www.keybr.com/) is not (just) a typing test

<p align="center">
    <img src="assets/screenshot.png" alt="screenshot" width="600"/>
</p>

It's the smartest way to learn touch typing and improve your typing speed.
On the surface, it looks pretty simple: it shows you a piece of text, and you type it out.
But the devil is in the details — keybr.com offers a few unique features:

* keybr.com tracks every single keystroke and computes statistics for each individual key.
* It automatically generates lessons that focus on your weakest keys.
* You can set your own target typing speed, and it tracks your progress toward that goal.
* It starts with a small set of the most frequent letters in your language.
* More letters are added once you reach the target speed with the current ones.
* It can even predict how many more lessons you will need to complete to reach your target speed.
* It provides a beautiful profile page with detailed graphs showing your learning progress.
* It offers plenty of modes and configuration options.

<p align="center">
    <img src="docs/assets/graph.png" alt="screenshot" width="600"/>
</p>

## Can I contribute?

Yes!

* **[Give us a ⭐️.](https://github.com/aradzie/keybr.com)** Help this project gain visibility and stand out.
* **[Report a bug.](https://github.com/aradzie/keybr.com/issues)** If something is not working, let us know.
* **[Suggest a feature.](https://github.com/aradzie/keybr.com/issues)** We are open to new ideas.
* **[Translate.](./docs/translations.md)** If you want to see keybr.com in your language.
* **[Getting started.](./docs/getting_started.md)** Launch a local instance of keybr.com, make a pull request.
* **[Add a keyboard.](docs/custom_keyboard.md)** Add a custom keyboard to keybr.com
* **[Add a language.](docs/custom_language.md)** Add a custom language to keybr.com
* **[Join our Discord server](https://discord.gg/gY4RA4enVH).** To discuss things in a less formal way.

## Self-Hosting

### Requirements

- Docker with Compose
- A domain or Tailscale hostname

### Setup

```bash
git clone https://github.com/codehia/keybr.com.git
cd keybr.com
```

Create a `.env` file:

```env
APP_URL=http://<your-hostname>:30044/

# Cookie
COOKIE_SECURE=false
COOKIE_DOMAIN=<your-hostname>

# Database (SQLite)
DATABASE_CLIENT=sqlite
DATABASE_FILENAME=/home/node/.local/state/keybr/keybr.db

# Google OAuth
AUTH_GOOGLE_CLIENT_ID=<your-client-id>
AUTH_GOOGLE_CLIENT_SECRET=<your-client-secret>

# Mail (required but unused if only using OAuth)
MAIL_DOMAIN=localhost
MAIL_KEY=none
```

Create the data directory and build:

```bash
mkdir -p .local/state/keybr
docker compose up --build -d
```

### Google OAuth Setup

1. Go to Google Cloud Console → APIs & Services → Credentials
2. Create an OAuth 2.0 Client ID
3. Add redirect URI: `http://<your-hostname>:30044/auth/oauth-callback/google`
4. Copy the client ID and secret into `.env`

### Notes

- Data is persisted in `./.local/state/keybr/`
- Ads are disabled by default in this fork

## License

Released under the GNU Affero General Public License v3.0.
