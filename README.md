# AWS Vault

[![Downloads](https://img.shields.io/github/downloads/99designs/aws-vault/total.svg)](https://github.com/99designs/aws-vault/releases)
[![Go Report Card](https://goreportcard.com/badge/github.com/99designs/aws-vault)](https://goreportcard.com/report/github.com/99designs/aws-vault)

AWS Vault is a tool to securely store and access AWS credentials in a development environment.

AWS Vault stores IAM credentials in your operating system's secure keystore and then generates temporary credentials from those to expose to your shell and applications. It's designed to be complementary to the AWS CLI tools, and is aware of your [profiles and configuration in `~/.aws/config`](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files).

The supported backends are:

* [macOS Keychain](https://support.apple.com/en-au/guide/keychain-access/welcome/mac)
* [Windows Credential Manager](https://support.microsoft.com/en-au/help/4026814/windows-accessing-credential-manager)
* Secret Service ([Gnome Keyring](https://wiki.gnome.org/Projects/GnomeKeyring), [KWallet](https://kde.org/applications/system/org.kde.kwalletmanager5))
* [KWallet](https://kde.org/applications/system/org.kde.kwalletmanager5)
* [Pass](https://www.passwordstore.org/)
* Encrypted file

Check out the [announcement blog post](https://99designs.com.au/tech-blog/blog/2015/10/26/aws-vault/) for more details.

## Installing

You can install aws-vault:
- by downloading the [latest release](https://github.com/99designs/aws-vault/releases)
- on macOS via [Homebrew Cask](https://github.com/caskroom/homebrew-cask) with `brew cask install aws-vault`
- on Linux via [Homebrew on Linux](https://docs.brew.sh/Homebrew-on-Linux) with `brew install aws-vault`
- on Windows via [choco](https://chocolatey.org/packages/aws-vault) with `choco install aws-vault`
- on Archlinux via the [AUR](https://aur.archlinux.org/packages/aws-vault/)
- by compiling with `go get github.com/99designs/aws-vault`


## Basic Usage

```bash
# Store AWS credentials for the "jonsmith" profile
$ aws-vault add jonsmith
Enter Access Key Id: ABDCDEFDASDASF
Enter Secret Key: %%%

# Execute a command (using temporary credentials)
$ aws-vault exec jonsmith -- aws s3 ls
bucket_1
bucket_2

# open a browser window and login to the AWS Console
$ aws-vault login jonsmith

# List credentials
$ aws-vault list
Profile                  Credentials              Sessions
=======                  ===========              ========
jonsmith                 jonsmith                 -
```
See the [USAGE](./USAGE.md) document for more help and tips.


## Security
```bash
$ aws-vault exec jonsmith -- env | grep AWS
AWS_VAULT=jonsmith
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=%%%
AWS_SECRET_ACCESS_KEY=%%%
AWS_SESSION_TOKEN=%%%
AWS_SECURITY_TOKEN=%%%
```

Notice in the above environment how a session token gets written out. This is because `aws-vault` uses Amazon's STS service to generate [temporary credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html) via the `GetSessionToken` or `AssumeRole` API calls. These expire in a short period of time, so the risk of leaking credentials is reduced.

The credentials are exposed to the subprocess in one of two ways:

 * Environment variables are written to the sub-process.

 * Local [EC2 Instance Metadata server](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) is started. This approach has the advantage that anything that uses Amazon's SDKs will automatically refresh credentials as needed, so session times can be as short as possible. The downside is that only one can run per host and because it binds to `169.254.169.254:80`, your sudo password is required.

The default is to use environment variables, but you can opt-in to the local instance metadata server with the `--server` flag on the `exec` command.

## Roles and MFA

[Best-practice](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#delegate-using-roles) is to [create Roles to delegate permissions](https://docs.aws.amazon.com/cli/latest/userguide/cli-roles.html). For security, you should also require that users provide a one-time key generated from a multi-factor authentication (MFA) device.

First you'll need to create the users and roles in IAM, as well as [setup an MFA device](https://docs.aws.amazon.com/IAM/latest/UserGuide/GenerateMFAConfigAccount.html). You can then [set up IAM roles to enforce MFA](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html#cli-configure-role-mfa).

Here's an example configuration using roles and MFA:

```ini
[default]
region = us-east-1

[profile jonsmith]
mfa_serial = arn:aws:iam::111111111111:mfa/jonsmith

[profile readonly]
role_arn = arn:aws:iam::22222222222:role/ReadOnly
source_profile = jonsmith

[profile admin]
role_arn = arn:aws:iam::22222222222:role/Administrator
source_profile = jonsmith
mfa_serial = arn:aws:iam::111111111111:mfa/jonsmith

[profile otheraccount]
role_arn = arn:aws:iam::333333333333:role/Administrator
source_profile = jonsmith
mfa_serial = arn:aws:iam::111111111111:mfa/jonsmith
```

When you use the `readonly` profile, aws-vault will use the credentials found in the source_profile `jonsmith`, and  generate temporary credentials for the `ReadOnly` role using STS AssumeRole. When you use the `prod-admin` profile, aws-vault will first use `GetSessionToken` to establish an MFA session before an AssumeRole for `Administrator`. This MFA session is stored in your keychain so you only have to enter your MFA once, and using `otheraccount` will not cause another MFA prompt.


## macOS Code Signing

The [macOS release builds](https://github.com/99designs/aws-vault/releases) are code-signed to avoid extra prompts in Keychain. You can verify this with:

    $ codesign --verify --verbose $(which aws-vault)

If you are developing or compiling the aws-vault binary yourself, you can [generate a self-signed certificate](https://support.apple.com/en-au/guide/keychain-access/kyca8916/mac) by accessing Keychain Access > Certificate Assistant > Create Certificate > Code Signing Certificate. You can then sign your binary with:

    $ go build .
    $ codesign --sign "Name of my certificate" ./aws-vault


## References and Inspiration

 * https://github.com/pda/aws-keychain
 * https://docs.aws.amazon.com/IAM/latest/UserGuide/MFAProtectedAPI.html
 * https://docs.aws.amazon.com/IAM/latest/UserGuide/IAMBestPractices.html#create-iam-users
 * https://github.com/makethunder/awsudo
 * https://github.com/AdRoll/hologram
 * https://github.com/realestate-com-au/credulous
 * https://github.com/dump247/aws-mock-metadata
 * https://boto.readthedocs.org/en/latest/boto_config_tut.html
