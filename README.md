# setup-mediawiki

A GitHub action to set up MediaWiki.

The action installs MediaWiki core, skins and extensions,
and exports a bunch of useful outputs for the other steps in your workflows.
In order to keep disruption to the rest of the system to a minimum,
it uses SQLite as the database backend (rather than MySQL/MariaDB or PostgreSQL),
and `php -S` as the web server (rather than e.g. Apache httpd or nginx).

## Usage

### Overview

```yaml
- uses: lucaswerkmeister/setup-mediawiki@v1
  with:
    # The version of MediaWiki (and skins and/or extensions) to check out.
    # Must be a valid ref in mediawiki/core.git, such as master (dev branch),
    # REL1_43 (release branch), or 1.43.0 (tag).
    version: 'master'

    # Extensions to clone and install, if any.
    # One per line (#-comments and blank lines allowed).
    # Each extension may be a plain name, in which case it is cloned from
    # Wikimedia Gerrit below mediawiki/extensions/, or a full Git URL to clone.
    extensions: ''

    # Skins to clone and install, if any.
    # One per line (#-comments and blank lines allowed).
    # Each skin may be a plain name, in which case it is cloned from Wikimedia Gerrit
    # below mediawiki/skins/, or a full Git URL to clone.
    skins: |
      Vector

    # Additional settings to append to LocalSettings.php.
    local-settings: ''

    # The user name of the initial MediaWiki admin user.
    # Also exposed as the admin-username output.
    admin-username: 'CI Wiki Admin'

    # The password of the initial MediaWiki admin user.
    # If blank, a random password is generated.
    # Also exposed as the admin-password output.
    admin-password: ''

    # The HTTP port under which to serve MediaWiki.
    port: '80'
```

### Outputs

<dl>
<dt>admin-username</dt>
<dd>The user name of the initial MediaWiki admin user.</dd>
<dt>admin-password</dt>
<dd>The password of the initial MediaWiki admin user.</dd>
<dt>server</dt>
<dd>The <code>$wgServer</code> variable, i.e. the protocol, hostname and port (origin) of the MediaWiki install. May be used to form URLs, though most of the time you probably want one of the other outputs instead.</dd>
<dt>script-path</dt>
<dd>The <code>$wgScriptPath</code> variable, i.e. the URL path below <code>$wgServer</code> to <code>api.php</code> and other entry points. May be used to form URLs, though most of the time you probably want one of the other outputs instead.</dd>
<dt>article-path</dt>
<dd>The <code>$wgArticlePath</code> variable, i.e. the URL pattern below <code>$wgServer</code> (with <code>$1</code> replaced by the title) used for internal wiki links. May be used to form URLs.</dd>
<dt>index-url</dt>
<dd>The full URL to the <code>index.php</code> (web) entry point.</dd>
<dt>api-url</dt>
<dd>The full URL of the <code>api.php</code> (<a href="https://www.mediawiki.org/wiki/Special:MyLanguage/API:Main_page">Action API</a>) entry point.</dd>
<dt>rest-url</dt>
<dd>The full URL of the <code>rest.php</code> (<a href="https://www.mediawiki.org/wiki/Special:MyLanguage/API:REST_API">REST API</a>) entry point.</dd>
<dt>install-directory</dt>
<dd>The absolute file path on the server of the MediaWiki install directory. May be used to run further maintenance scripts from inside it.</dd>
</dl>

### Simple example

```yaml
- uses: lucaswerkmeister/setup-mediawiki@v1
  id: setup-mediawiki
- run: |
    curl "$WIKI_API_URL?action=query&meta=siteinfo&format=json&formatversion=2"
  env:
    WIKI_API_URL: ${{ steps.setup-mediawiki.outputs.api-url }}
```

This will install the current `master` version of MediaWiki,
with the Vector skin and no extensions,
and fetch its siteinfo API response via cURL.

### Release version, extensions and skins

```yaml
- uses: lucaswerkmeister/setup-mediawiki@v1
  id: setup-mediawiki
  with:
    version: REL1_43
    extensions: |
      Cite
      Echo
      Gadgets
      MobileFrontend
    skins: |
      MinervaNeue
      Vector
- run: |
    curl "$WIKI_INDEX_URL?title=Special:Version&useformat=mobile"
  env:
    WIKI_INDEX_URL: ${{ steps.setup-mediawiki.outputs.index-url }}
```

This will install the latest MediaWiki 1.43 version,
with a few extensions and the mobile frontend + skin,
and show the mobile version of the Special:Version page.

### Custom settings and maintenance scripts

```yaml
- uses: lucaswerkmeister/setup-mediawiki
  id: setup-mediawiki
  with:
    extensions: |
      OAuth
    local-settings: |
      $wgGroupPermissions['sysop']['mwoauthproposeconsumer'] = true;
      $wgGroupPermissions['sysop']['mwoauthupdateownconsumer'] = true;
      $wgGroupPermissions['sysop']['mwoauthmanageconsumer'] = true;
      $wgGroupPermissions['sysop']['mwoauthsuppress'] = true;
      $wgGroupPermissions['sysop']['mwoauthviewsuppressed'] = true;
      $wgGroupPermissions['sysop']['mwoauthviewprivate'] = true;
      $wgGroupPermissions['sysop']['mwoauthmanagemygrants'] = true;
- run: |
    cd -- "$MEDIAWIKI_INSTALL_DIR"
    php maintenance/run.php resetUserEmail --no-reset-password "$MEDIAWIKI_USERNAME" 'mail@localhost'
    json=$(php maintenance/run.php $PWD/extensions/OAuth/maintenance/createOAuthConsumer.php \
        --user="$MEDIAWIKI_USERNAME" \
        --name='CI consumer' \
        --version="$(printf '%(%s)T')" \
        --description='OAuth consumer for CI' \
        --callbackUrl='http://localhost:12345/oauth/callback' \
        --grants=editpage \
        --approve \
        --jsonOnSuccess
    )
    jq -r '"client-id=" + .key + "\nclient-secret=" + .secret' <<< "$json" >> "$GITHUB_OUTPUT"
  shell: bash
  env:
    MEDIAWIKI_USERNAME: ${{ steps.setup-mediawiki.outputs.admin-username }}
    MEDIAWIKI_INSTALL_DIR: ${{ steps.setup-mediawiki.outputs.install-directory }}
  id: create-oauth-client
```

This will install MediaWiki with the OAuth extension,
configure the extension to allow administrators to manage OAuth consumers,
create a test consumer via a maintenance script,
and expose that consumer’s ID and secret via outputs for later steps in the workflow.

### Installing a custom extension

```yaml
- uses: lucaswerkmeister/setup-mediawiki
  id: setup-mediawiki
- uses: actions/checkout@v4
  with:
    path: ${{ steps.setup-mediawiki.outputs.install-directory }}/extensions/MyExtension
- run: |
    cd -- "$MEDIAWIKI_INSTALL_DIR"
    cat >> LocalSettings.php << 'EOF'
    wfLoadExtension( 'MyExtension' );
    EOF
    composer update
    php maintenance/run.php update --quick
```

This will install MediaWiki without any extensions,
then clone and install the extension from the current GitHub repository separately.
The clone must be separate so that the checkout action can use the current ref (e.g. from a pull request),
and the install must be separate because setup-mediawiki expects the wiki to be functional immediately after installation –
if `wfLoadExtension( 'MyExtension' )` was included in the `local-settings` input,
then the action would crash when trying to export its output variables because `MyExtension` isn’t available yet.

## Prior work

This action is mostly unrelated to [ProfessionalWiki/setup-mediawiki](https://github.com/ProfessionalWiki/setup-mediawiki),
though I did look at it for inspiration (before deciding to roll my own).

## License

[Blue Oak Model License, version 1.0.0](https://blueoakcouncil.org/license/1.0.0).
