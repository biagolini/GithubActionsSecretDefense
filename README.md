# GitHub Actions Secret Defense

**Author:** Carlos Biagolini-Jr.

**LinkedIn:** [https://www.linkedin.com/in/biagolini/](https://www.linkedin.com/in/biagolini/)

**Medium:** [https://medium.com/@biagolini](https://medium.com/@biagolini)

**Companion article:** [https://medium.com/@biagolini/41cee1adac12](https://medium.com/@biagolini/41cee1adac12)

> **Didactic notice:** This project was developed as a practical example to accompany a blog post. The architecture decisions made here were chosen to meet an educational objective. They illustrate concepts clearly, not necessarily in the most production-ready way. Feel free to draw inspiration from these ideas, but remember to evaluate the limitations and constraints of your own business before adopting any of them. This project is released under the [MIT License](./LICENSE): you are free to copy, modify, and use it as you wish, but it comes with no warranties and the author takes no responsibility for its use in any environment.

## Overview

This is the service (victim) repository for the secret exfiltration defense article. It is a minimal Python project: a small `calculator` module with pytest tests, packaged as a container. On top of that minimal project, it ships five attack workflows that a collaborator with the **Write** role could use to extract a GitHub secret or steal an AWS role session from a pipeline.

The article shows how an organization administrator stops these attacks with controls that live outside this repository: an [organization push ruleset](https://docs.github.com/en/organizations/managing-organization-settings/creating-rulesets-for-repositories-in-your-organization) on `.github/`, a branch ruleset on `main`, [workflow execution protections](https://docs.github.com/en/actions/how-tos/administer/control-workflow-execution), [SHA-pinned actions](https://docs.github.com/en/organizations/managing-organization-settings/disabling-or-limiting-github-actions-for-your-organization), an AWS IAM role whose trust policy checks individual [GitHub OIDC claims](https://docs.github.com/en/actions/reference/security/oidc), and a [resource control policy](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html) in AWS Organizations.

This repository holds only the code shown in the article. The GitHub organization and AWS account controls that make the design safe are configured separately, as the article describes.

### Attack workflows

All attack workflows carry a `.insecure` suffix so GitHub does not run them in this reference copy. To reproduce one, recreate it (without the suffix) in your own test repository once you have the matching GitHub secret and AWS role in place.

- `01-leak-secret.yml.insecure` — leaks `DEMO_SECRET` by printing its base64 form on `push`, which log masking does not catch.
- `02-dispatch-leak.yml.insecure` — same payload, triggered by `workflow_dispatch`, a separate entry point from `push`.
- `03.01-external-call.yml.insecure` and `03.02-external-call.yml.insecure` — runs code from another repository: the caller (part 2, lives here) points `uses:` at a reusable workflow (part 1, lives in [GithubActionsSecretAttackerWorkflows](https://github.com/biagolini/GithubActionsSecretAttackerWorkflows)) and passes the secret by name.
- `04-assume-role.yml.insecure` — assumes the AWS role from a branch and prints the temporary session three ways (raw, single base64, double base64) to show what GitHub log masking does and does not catch.
- `05-insecure-ci.yml.insecure` — the vulnerable CI pipeline that holds the role and runs `pytest`, which a malicious test (`tests/test_calculator.py.insecure`) uses to leak both the role session and the secret without editing any workflow.

## Prerequisites

- A GitHub organization on the **Team** plan (or Enterprise Cloud) with two accounts: an owner and a member with the **Write** role on a private copy of this repository.
- An organization-level secret `DEMO_SECRET` with a made-up value, shared with the repository.
- An AWS account (use a throwaway account) with the GitHub OIDC provider `token.actions.githubusercontent.com` configured, and an IAM role with a loose trust policy, referenced by a secret `AWS_ROLE_ARN`. Attacks 4 and 5 use this role.
- The [hardened central workflow](https://github.com/biagolini/GithubActionsPlatformWorkflows) is used by the defended pipeline (Part 3 of the article).

## Reproducing the attacks

Each attack is a separate workflow. To run one, recreate its content in `.github/workflows/` under a name of your choice (for example `production.yml`), as the article instructs, then push or dispatch it as the Write member. Decode any `LEAK_SECRET:` or `LEAK_AWS:` line from the log with two base64 passes:

```bash
echo "<string-after-the-colon>" | base64 -d | base64 -d
```

The attacks are described step by step in the companion article.

## Project Structure

```
.
├── .github/
│   └── workflows/
│       ├── 01-leak-secret.yml.insecure        # Attack 1: leak secret on push
│       ├── 02-dispatch-leak.yml.insecure      # Attack 2: leak via workflow_dispatch
│       ├── 03.01-external-call.yml.insecure   # Attack 3, part 1: attacker reusable workflow (reference)
│       ├── 03.02-external-call.yml.insecure   # Attack 3, part 2: caller in this repo
│       ├── 04-assume-role.yml.insecure        # Attack 4: assume role and exfiltrate session
│       └── 05-insecure-ci.yml.insecure        # Attack 5: vulnerable CI that a malicious test abuses
├── src/
│   ├── __init__.py
│   └── calculator.py                          # minimal app under test
├── tests/
│   ├── test_calculator.py                     # legitimate tests
│   └── test_calculator.py.insecure            # malicious test used by attack 5
├── Dockerfile
├── pytest.ini
├── requirements.txt
├── LICENSE
└── README.md
```

## Cleanup

After testing, remove what you created:

- Delete this repository (or its workflows) from your test organization.
- Delete the IAM role and the ECR repository used by attacks 4 and 5.
- Rotate `DEMO_SECRET` and any AWS credential that was readable during the test.

## License

This project is open source and available under the [MIT License](./LICENSE).
