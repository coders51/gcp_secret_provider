# GCP Secret Provider

[![Module Version](https://img.shields.io/hexpm/v/gcp_secret_provider.svg)](https://hex.pm/packages/gcp_secret_provider)
[![Hex Docs](https://img.shields.io/badge/hex-docs-lightgreen.svg)](https://hexdocs.pm/gcp_secret_provider/)
[![Total Download](https://img.shields.io/hexpm/dt/gcp_secret_provider.svg)](https://hex.pm/packages/gcp_secret_provider)

This is a runtime config provider for secrets on GCP. It will replace config in your app with secrets stored in [Google's secret manager API](https://cloud.google.com/secret-manager) on app boot.

That means secrets are encrypted at rest and you can cycle or change a secret in the cloud console and re-start (note: _not_ redeploy) the app to get the new secrets. It also means you don't have to put all of your secrets into your build server (circle, travis or github actions) and you do not have to check the secrets into source control. Access to the secrets can be tightly controlled by IAM policies in GCP without affecting anyone's ability to deploy the app if needed.

To work it needs one thing, a service account with the [Secret Manager Secret Accessor role](https://cloud.google.com/secret-manager/docs/access-control). We recommend that you do that in `releases.exs` so that it is runtime config:

```elixir
config(:gcp_secret_provider, service_account: System.fetch_env!("GOOGLE_APPLICATION_CREDENTIALS"))
```

## Example

In your `mix.exs` where your mix release is configured add this:

```elixir
def project do
  [
    ...
    releases: releases(),
    ...
  ]
end

def releases() do
  [
    my_release_name: [
      ...
      config_providers: [{GcpSecretProvider, %{project: "my_google_project_name-12345"}}],
      ...
    ]
  ]
end
```

Then in your `config/prod.exs` or your `releases.exs` you can write this:

```elixir
config(:db, Db.Repo,
  username: {"GAE_SECRET", :string, "PG_DATABASE_USER"},
  password: {"GAE_SECRET", :string, "PG_DATABASE_PASSWORD"},
  database: {"GAE_SECRET", :string, "PG_DATABASE_DB"},
  socket_dir: {"GAE_SECRET", :string, "PG_DATABASE_SOCKET_DIR"},
  port: {"GAE_SECRET", :integer, "PG_DATABASE_PORT", "latest"}
)
```

The format is `{"GAE_SECRET", type, name_of_secret, version}` with version optional and defaulting to the latest secret version. When the app boots all config will be parsed, and replaced with the actual secrets. We will attempt to cast the given secret into the given type, current accepted types are:

```elixir
:string
:integer
```

### The 🐥 and the 🥚

There is a slight issue here, in that the service account required to access the secrets is itself (in theory) a secret. So what can we do to ensure that this is kept somewhat secure? After all if they can access that service account they can access the secrets. Here is the approach I took your's may differ**.

1. Password encrypt the required service account with gpg: `gpg --symmetric --cipher-algo AES256 my_service_account_key.json`
2. Give it a secure passcode (ideally generated by a password manager)
3. Check the file into source control.
4. Put the passcode for decryption into the build server as secret. [Here's how in github actions](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets)
5. Add a build step which decrypts the file using the secret and puts it in the env under a GOOGLE_APPLICATION_CREDENTIALS key.

In my case I was deploying to GAE, and the only way to get an env into GAE is putting it into an `app.yaml` file. So in my `app.yaml` file I added this:

```yaml
# Imports the default service account as a secret during deployment.
# This is then used in the config provider to get the rest of the secrets.
includes:
  - secrets.yaml
```

Then I had a build step which created that secrets.yaml file and put in the decrypted service account:

```bash
#!/usr/bin/env bash
# Exit immediately if a command exits with a non-zero status.
set -e

touch secrets.yaml

DECODED=$(gpg --quiet --batch --yes --decrypt --passphrase="$SECRET_PASSPHRASE" default_service_account.json.gpg)

# Copies the service account into our secrets.yaml file so when we deploy it gets set as an env var
cat <<EOF > secrets.yaml
env_variables:
  GOOGLE_APPLICATION_CREDENTIALS: '$DECODED'
EOF
```

Now you may ask why might I do all this rather than just get all the secrets in a build step? Well you can just get them from the secret manager as a build step, but this allows the secrets to be refreshed on app boot, meaning you don't have to redeploy to get new secrets.


## Installation

If [available in Hex](https://hex.pm/docs/publish), the package can be installed
by adding `gcp_secret_provider` to your list of dependencies in `mix.exs`:

*NOTE* If you are using an umbrella project, the dependancy can't be in the root `mix.exs`, but must be inside one of the apps that the release is building.


```elixir
def deps do
  [
    {:gcp_secret_provider, "~> 0.1.2"}
  ]
end
```

Documentation can be generated with [ExDoc](https://github.com/elixir-lang/ex_doc)
and published on [HexDocs](https://hexdocs.pm). Once published, the docs can
be found at [https://hexdocs.pm/gcp_secret_provider](https://hexdocs.pm/gcp_secret_provider).


** If you are using GAE it has a default service account, so you may be fine with that, and not have to supply a service account at all. I couldn't get it to work.
