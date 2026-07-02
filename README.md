# 🌼 Configure Huawei Cloud short-lived credentials via OIDC for GitHub Actions

[![🧪 Testing](https://github.com/vbem/configure-huawei-cloud-credentials/actions/workflows/test.yml/badge.svg)](https://github.com/vbem/configure-huawei-cloud-credentials/actions/workflows/test.yml)
[![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/vbem/configure-huawei-cloud-credentials?label=Release&logo=github)](https://github.com/vbem/configure-huawei-cloud-credentials/releases)
[![Marketplace](https://img.shields.io/badge/GitHub%20Actions-Marketplace-blue?logo=github)](https://github.com/marketplace/actions/configure-huawei-cloud-credentials)

## About

⚠️⚠️⚠️ ***This action will become publicly available in August 2026, after Huawei Cloud releases the IAM/STS v5 OIDC APIs.***

<div align="center">
  <img src="https://docs.github.com/assets/cb-63262/mw-1440/images/help/actions/oidc-architecture.webp" width="600" alt="OIDC Architecture">
</div>

Huawei Cloud does not currently provide an official GitHub Action for OIDC-based credentials. This action fills that gap by configuring [temporary credentials](https://support.huaweicloud.com/usermanual-iam5/iam_01_1236.html) (**Access Key ID / Secret Access Key / Security Token**) for GitHub Actions, using a [GitHub OIDC token](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-cloud-providers) exchanged with Huawei Cloud Security Token Service (STS). Workflows can access Huawei Cloud resources without storing long-lived *Access Key ID / Secret Key* pairs in GitHub Secrets. Comparable actions for other clouds and platforms include:

- [`aws-actions/configure-aws-credentials`](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions)
- [`azure/login`](https://github.com/marketplace/actions/azure-login)
- [`google-github-actions/auth`](https://github.com/marketplace/actions/authenticate-to-google-cloud)
- [`aliyun/configure-aliyun-credentials-action`](https://github.com/marketplace/actions/configure-alibaba-cloud-credentials-action-for-github-actions)
- [`everpcpc/tencentcloud-oidc-auth`](https://github.com/marketplace/actions/authenticate-to-tencent-cloud)
- [`hashicorp/vault-action`](https://github.com/marketplace/actions/hashicorp-vault)
- [`jfrog/setup-jfrog-cli`](https://github.com/marketplace/actions/setup-jfrog-cli)
- [`pypa/gh-action-pypi-publish`](https://github.com/marketplace/actions/pypi-publish)

## Example Usage

```yaml
jobs:
  example:
    runs-on: ubuntu-slim
    timeout-minutes: 1
    defaults: {run: {shell: bash}}
    permissions: {id-token: write, contents: read}

    steps:
      - name: 🔑 Generate Huawei Cloud temporary credentials
        id: creds
        uses: vbem/configure-huawei-cloud-credentials@main
        with:
          provider-urn: iam::<account-id>:oidcProvider:<provider-name>
          agency-urn: iam::<account-id>:agency:<agency-name>

      - name: 🔍 Print outputs of previous step
        env: {STEP_OUTPUTS: "${{ toJson(steps.creds.outputs) }}"}
        run: jq -C <<<<"$STEP_OUTPUTS"

      - name: 🖥️ Setup Huawei Cloud KooCLI for testing
        uses: vbem/setup-hcloud@main

      - name: 🧪 Test temporary credentials using KooCLI
        run: |-
          hcloud sts GetCallerIdentity --cli-region=cn-east-3 \
            --cli-access-key="${HUAWEICLOUD_SDK_AK}" \
            --cli-secret-key="${HUAWEICLOUD_SDK_SK}" \
            --cli-security-token="${HUAWEICLOUD_SDK_SECURITY_TOKEN}" \
            | jq -C
```

Notes:

1. The workflow must [grant `id-token: write` permission](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-cloud-providers#adding-permissions-settings) so GitHub Actions can issue an OIDC token for this action.
1. This [composite action](https://docs.github.com/en/actions/concepts/workflows-and-actions/custom-actions) requires `bash`, `curl`, and `jq`. These tools are [pre-installed](https://github.com/actions/runner-images/tree/main/images) on the official Unix-like GitHub-hosted runners.

## Inputs

ID | Type | Default | Description
--- | --- | --- | ---
`provider-urn` | String | Required | The Huawei Cloud IAM v5 OIDC provider URN, in the format `iam::<account-id>:oidcProvider:<provider-name>`.
`agency-urn` | String | Required | The Huawei Cloud IAM v5 agency URN to assume, in the format `iam::<account-id>:agency:<agency-name>`.
`audience` | String | `sts.huaweicloud.com` | The audience for the GitHub OIDC token.
`session-name` | String | `GitHubActions` | The agency session name.
`duration-seconds` | Number | `900` | The agency session duration, from `900` seconds (15 minutes) to `43200` seconds (12 hours).
`export-env` | Boolean | `true` | Whether to export the [temporary credentials as environment variables](https://github.com/huaweicloud/huaweicloud-sdk-java-v3#241-environment-variables-top) for subsequent workflow steps.
`env-ak-name` | String | `HUAWEICLOUD_SDK_AK` | The environment variable name used to export the Access Key ID.
`env-sk-name` | String | `HUAWEICLOUD_SDK_SK` | The environment variable name used to export the Secret Access Key.
`env-st-name` | String | `HUAWEICLOUD_SDK_SECURITY_TOKEN` | The environment variable name used to export the Security Token.
`sts-region` | String | `cn-north-4` | The [Huawei Cloud STS API region](https://support.huaweicloud.com/api-iam5/iam_02_1101.html) to use.

## Outputs

ID | Type | Description | Example
--- | --- | --- | ---
`ak` | String | Access Key ID for the temporary credential. | `HSTANO...........`
`sk` | String | Secret Access Key for the temporary credential. | `EoWCQrr...........`
`st` | String | Security Token for the temporary credential. | `hQpjbi1...........`
`expiration` | Datetime | Expiration time for the temporary credential, in RFC 3339 format. | `2026-09-07T03:27:51.158Z`
`urn` | String | The assumed agency URN. | `sts::<account-id>::assumed-agency:<agency-name>/<session-name>`
`id` | String | The assumed agency ID. | `<agency-id>:<session-name>`

## Huawei Cloud IAM v5 Configuration

Before using this action, set up an OIDC provider and agency in Huawei Cloud [IAM v5](https://support.huaweicloud.com/productdesc-iam5/iam_01_1105.html). Legacy [IAM v3](https://support.huaweicloud.com/iam/index.html) does not support OIDC-based agency federation and cannot be used with this action.

For [***IAM Identity Provider***](https://console.huaweicloud.com/iam5/#/idp), the following settings are recommended:

Name | Recommended Value | Description
--- | --- | ---
Type | `OIDC` | The identity provider type.
Identity Provider Name | `github_com` | A name that identifies github.com or a GHES instance as the provider.
Identity Provider URL | `https://token.actions.githubusercontent.com` | The [OIDC token issuer for github.com](https://docs.github.com/en/actions/reference/security/oidc). For [GitHub Enterprise Server (GHES)](https://docs.github.com/en/enterprise-server@latest/actions/reference/security/oidc), use `https://GHES_HOSTNAME/_services/token`.
Audience | `sts.huaweicloud.com` | The [OIDC token audience](https://docs.github.com/en/actions/reference/security/oidc). It must match the `audience` input of this action.
Description | The URL of this action | Helps identify how this provider is used.

For [***IAM Agency***](https://support.huaweicloud.com/usermanual-iam5/iam_01_0915.html), the following settings are recommended:

Name | Recommended Value | Description
--- | --- | ---
Agency Name | `gh-<usage-desc>` | A name that identifies this agency's purpose, e.g. `gh-terraform-foobar-prod`.
Agency Type | Custom trust policy | A custom trust policy can bind the agency to a specific OIDC provider and define flexible trust conditions.
Description | The URL of the OIDC identity provider | Helps identify how this agency is used.
Authorized Policies | As needed | Attach only the least-privilege [policies](https://support.huaweicloud.com/usermanual-iam5/iam_01_1159.html) required for your use case.

An [IAM agency's ***Trust Policy***](https://support.huaweicloud.com/usermanual-iam5/iam_01_0915.html#section2) controls who can assume the agency and under what conditions. The sample below allows GitHub Actions workflows in a specific repository to assume the agency. You can further restrict the [`oidc:sub` claim](https://docs.github.com/en/actions/reference/security/oidc#example-subject-claims) by organization, repository, branch, tag, environment, or other workflow context. Note that GitHub's [*Immutable Subject Claims* feature](https://docs.github.com/en/actions/reference/security/oidc#immutable-subject-claims) may change the `oidc:sub` format on github.com, but not on GHES, for repositories created, renamed, or transferred after July 15, 2026. Existing repositories can enable or disable this feature at the repository level in the GitHub UI (Settings > Actions > OIDC > Use immutable subject claim).

```jsonc
{
  "Version": "5.0",
  "Statement": [
    {
      "Action": ["sts:agencies:assumeWithOIDC"],
      "Effect": "Allow",
      "Condition": {
        "StringMatch": {
          // `oidc:sub` supports org/repo/branch/tag/environment/etc.
          "oidc:sub": "repo:<github-owner-or-org>/<github-repo-id>:*"
        },
        "StringEquals": {
          // `oidc:aud` must match `audience`
          "oidc:aud": ["sts.huaweicloud.com"],
          // `oidc:iss` must match the OIDC token issuer
          "oidc:iss": ["https://token.actions.githubusercontent.com"]
        }
      },
      "Principal": {
        // `Federated` must match the OIDC provider URN
        "Federated": ["<OIDC-provider-URN>"]
      }
    }
  ]
}
```
