# antijante/packagist

Private Composer index for Antijante WordPress and WooCommerce projects, served at
<https://antijante.github.io/packagist>.

The index itself is public, but each paid plugin lives in its own **private** mirror
repo under `antijante`, so downloading a plugin requires a GitHub token with read
access to that repo.

Free plugins come from [wpackagist.org](https://wpackagist.org); they are not mirrored here.

## Use it in a project

`composer.json`:

```json
{
  "repositories": [
    { "type": "composer", "url": "https://antijante.github.io/packagist" },
    { "type": "composer", "url": "https://wpackagist.org", "only": ["wpackagist-plugin/*", "wpackagist-theme/*"] }
  ]
}
```

Authenticate Composer once per machine or CI job with a token that can read the mirrors:

```sh
composer config -g github-oauth.github.com <token>
```

Then `composer require antijante/<plugin>`.

## Add a paid plugin (once you have a license)

1. Create a private repo `antijante/<plugin-slug>` with a `composer.json`:

    ```json
    {
      "name": "antijante/<plugin-slug>",
      "type": "wordpress-plugin",
      "description": "Plugin name",
      "homepage": "https://vendor.example/plugin"
    }
    ```

2. Add `.github/workflows/build.yml` that checks the vendor daily, commits and tags new
   versions, then triggers a rebuild here. The reusable updaters in
   [`antijante/github-action-update-plugins`](https://github.com/antijante/github-action-update-plugins)
   (our copy of Genero's MIT-licensed repo) cover WooCommerce.com, ACF, Gravity Forms,
   FacetWP, WPML, EDD, Freemius and others.

    ```yml
    name: Build
    on:
      workflow_dispatch:
      schedule:
        - cron: '5 4 * * *'
    jobs:
      build:
        uses: antijante/github-action-update-plugins/.github/workflows/wccom-update.yml@main
        secrets:
          ACCESS_TOKEN: ${{ secrets.WCCOM_ACCESS_TOKEN }}
          ACCESS_TOKEN_SECRET: ${{ secrets.WCCOM_ACCESS_TOKEN_SECRET }}
        with:
          slug: '<plugin-slug>'
      update-index:
        needs: build
        if: needs.build.outputs.updated == 'true'
        uses: antijante/packagist/.github/workflows/update.yml@main
        secrets:
          token: ${{ secrets.PACKAGIST_DISPATCH_TOKEN }}
    ```

3. Add the license secrets to the mirror repo (or as org secrets scoped to it).

4. Add the mirror to `satis.json`:

    ```json
    { "type": "vcs", "url": "https://github.com/antijante/<plugin-slug>.git" }
    ```

5. Run the mirror's Build workflow once to fetch the first version.

## Secrets

| Secret | Where | Purpose |
| --- | --- | --- |
| `MIRROR_READ_TOKEN` | this repo | Lets the index build read private mirrors (Contents: read on all mirrors). Required once the first private mirror is added. |
| `PACKAGIST_DISPATCH_TOKEN` | org, scoped to mirror repos | Optional. Lets mirrors trigger an instant rebuild (Contents: write on this repo). |

## Rebuilds

- On every push to `main`.
- When a mirror dispatches after a release.
- Twice a day on schedule (04:30 and 16:30 UTC), as a safety net.
- Manually: `gh workflow run build.yml --repo antijante/packagist`.
