# 🌼 Configure Huawei Cloud Credentials via OIDC for GitHub Actions

[![🧪 Testing](https://github.com/vbem/configure-huawei-cloud-credentials/actions/workflows/test.yml/badge.svg)](https://github.com/vbem/configure-huawei-cloud-credentials/actions/workflows/test.yml)
[![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/vbem/configure-huawei-cloud-credentials?label=Release&logo=github)](https://github.com/vbem/configure-huawei-cloud-credentials/releases)
[![Marketplace](https://img.shields.io/badge/GitHub%20Actions-Marketplace-blue?logo=github)](https://github.com/marketplace/actions/configure-huawei-cloud-credentials)

⚠️⚠️⚠️ ***This action will become publicly available in August 2026, following the release of HW Cloud IAM/STS v5 API.***

## About

As a supplement to the lack of official action from Huawei Cloud, this action configures [temporary credentials](https://support.huaweicloud.com/usermanual-iam5/iam_01_1236.html) (**Access Key ID / Secret Access Key / Security Token**) for GitHub Actions by [exchanging a GitHub OIDC token](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-cloud-providers) with Huawei Cloud STS service. It lets workflows access Huawei Cloud resources without storing long-lived *Access Key ID / Secret Key* pairs in GitHub Secrets. This action is similar to [`aws-actions/configure-aws-credentials`](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions) and [`aliyun/configure-aliyun-credentials-action`](https://github.com/marketplace/actions/configure-alibaba-cloud-credentials-action-for-github-actions).

## Example usage

```yaml
jobs:
  example:
    runs-on: ubuntu-slime
    timeout-minutes: 1
    defaults: {run: {shell: bash}}
    permissions: {id-token: write, contents: read}

    steps:
      - name: 🌼 Configure HW Cloud temporary credentials
        id: creds
        uses: vbem/configure-huawei-cloud-credentials@main
        with:
          provider-urn: iam::<account-id>:oidcProvider:<provider-name>
          agency-urn: iam::<account-id>:agency:<agency-name>

      - name: 🔍 Print the action outputs
        env: {STEP_OUTPUT: "${{ toJson(steps.creds.outputs) }}"}
        run: jq -C <<<<"$STEP_OUTPUT"

      - name: 🖥️ Setup HW Cloud KooCLI for testing
        uses: vbem/setup-hcloud@main

      - name: 🧪 Test temporary credentials using KooCLI
        run: |-
          hcloud sts GetCallerIdentity --cli-region=cn-east-3 \
            --cli-access-key="${HUAWEICLOUD_SDK_AK}" \
            --cli-secret-key="${HUAWEICLOUD_SDK_SK}" \
            --cli-security-token="${HUAWEICLOUD_SDK_SECURITY_TOKEN}" \
            | jq -C
```

## Inputs

ID | Type | Default | Description
--- | --- | --- | ---
`provider-urn` | String | Required | The OIDC provider URN in Huawei Cloud IAM v5, in format of `iam::<account-id>:oidcProvider:<provider-name>`.
`agency-urn` | String | Required | The agency URN to assume in Huawei Cloud IAM v5, in format of `iam::<account-id>:agency:<agency-name>`.
`audience` | String | `sts.huaweicloud.com` | The audience for the GitHub OIDC token.
`session-name` | String | `GitHubActions` | The session name for agency assuming.
`duration-seconds` | Number | `900` | The duration seconds for agency assuming, between `900` (15 mins) and `43200` (12 hrs).
`export-env` | Boolean | `true` | Whether to export the temporary credentials as environment variables for subsequent workflow steps.
`env-ak-name` | String | `HUAWEICLOUD_SDK_AK` | The environment variable name for Access Key ID to export.
`env-sk-name` | String | `HUAWEICLOUD_SDK_SK` | The environment variable name for Secret Access Key to export.
`env-st-name` | String | `HUAWEICLOUD_SDK_SECURITY_TOKEN` | The environment variable name for Security Token to export.
`sts-region` | String | `cn-north-4` | The [region of Huawei Cloud STS API](https://support.huaweicloud.com/api-iam5/iam_02_1101.html) for requesting.

## Outputs

ID | Type | Description | Example
--- | --- | --- | ---
`ak` | String | Access Key ID of the temporary credential. | `HSTANO...........`
`sk` | String | Secret Access Key of the temporary credential. | `EoWCQrr...........`
`st` | String | Security Token of the temporary credential. | `hQpjbi1...........`
`expiration` | Datetime | Expiration time of the temporary credential in ISO 8601 format. | `2022-09-07T03:27:51.158Z`
`urn` | String | URN of the assumed agency. | `sts::<account-id>::assumed-agency:<agency-name>/<session-name>`
`id` | String | Unique ID of the assumed agency. | `<agency-id>:<session-name>`

## HW Cloud IAM v5 Configuration

Before using this action, you need to configure the OIDC provider and agency in Huawei Cloud IAM v5.

For [Identity Provider](https://console.huaweicloud.com/iam5/#/idp), the following settings are recommended:

Name | Recommended Value | Description
--- | --- | ---
Type | `OIDC` | The identity provider type.
Identity Provider Name | `github_com` | The name to distinguish GitHub servers.
Identity Provider URL | `https://token.actions.githubusercontent.com` | The [OIDC token issuer of the GitHub server](https://docs.github.com/en/actions/reference/security/oidc).
Audience | `sts.huaweicloud.com` | The [audience for the OIDC token](https://docs.github.com/en/actions/reference/security/oidc), which should be consistent with the `audience` input of this action.
Description | The URL of this action | It can help you quickly identify the usage of this provider in the future.

For [Agency](https://support.huaweicloud.com/usermanual-iam5/iam_01_0915.html), the following settings are recommended:

Name | Recommended Value | Description
--- | --- | ---
Agency Name | `gh-<usage-desc>` | The name to distinguish the usage of this agency, e.g. `gh-terraform-foobar-prod`.
Agency Type | Custom trust policy | A "Trust Agency" allows binding to a specific OIDC provider and defining the trust policy with more flexibility.
Description | The URL of the OIDC Identity provider | It can help you quickly identify the usage of this agency in the future.
Authorized Poilies | Usage based | Attach the least-privilege [policies](https://support.huaweicloud.com/usermanual-iam5/iam_01_1159.html) based on your usage.
Trust Policy | See below | See below

The [trust policy](https://support.huaweicloud.com/usermanual-iam5/iam_01_0915.html#section2) controls who can assume this agency and under what conditions. Following is a sample trust policy, which allows GitHub Actions workflows in a specific repository to assume this agency. You can further restrict the permissions by limiting the [`oidc:sub` condition](https://docs.github.com/en/actions/reference/security/oidc#example-subject-claims) or adding other conditions based on your needs.

```json
{
  "Version": "5.0",
  "Statement": [
    {
      "Action": ["sts:agencies:assumeWithOIDC"],
      "Effect": "Allow",
      "Condition": {
        "StringLike": {
          "oidc:sub": "repo:<github-owner-or-org>/<github-repo-id>:*"
        },
        "StringEquals": {
          "oidc:aud": ["sts.huaweicloud.com"],
          "oidc:iss": ["https://token.actions.githubusercontent.com"]
        }
      },
      "Principal": {
        "Federated": ["<OIDC-provider-URN>"]
      }
    }
  ]
}
```

## Notes

1. This action is only compatible with latest [HW Cloud IAM/STS v5](https://support.huaweicloud.com/productdesc-iam5/iam_01_1107.html), the legacy [IAM v3](https://support.huaweicloud.com/iam/index.html) do not support OIDC-based agency federation.
2. The workflow must [grant `id-token: write` permission](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-cloud-providers#adding-permissions-settings) so GitHub Actions can issue an OIDC token for this action.
3. This [composite action](https://docs.github.com/en/actions/concepts/workflows-and-actions/custom-actions) requires `bash`, `curl`, and `jq` to be available in the runner environment. The official GitHub-hosted runners have these tools pre-installed.
