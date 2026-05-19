---
title: Deploying Remotion Lambda with Terraform
slug: remotion-lambda-terraform
description: Replacing @remotion/lambda's CLI deployer with a Terraform module — plus the function-name and 250 MB layer traps that bite in production.
tags:
  - aws
  - terraform
  - remotion
  - lambda
  - code
added: "May 19 2026"
---

[Remotion](https://www.remotion.dev/) is a React-based video framework. Its `@remotion/lambda` package gives you distributed rendering on AWS — one orchestrator Lambda fans out frames to up to 200 renderer Lambdas, then stitches the result back together. The official deployment story is a CLI (`npx remotion lambda functions deploy`) that imperatively creates a Lambda, an S3 bucket, an IAM role and a log group on your behalf.

That's great for a greenfield project. It's a non-starter if you already manage AWS through Terraform and want video rendering to live in the same plan/apply loop as the rest of your infra.

This post walks through how I rebuilt the Remotion Lambda setup as a Terraform module, and the two non-obvious things that broke on me in production: the hard-coded function name during version upgrades, and the 250 MB layer ceiling once you bolt Datadog on. Benjamin Kunkel's [CDK writeup](https://bndkt.com/blog/2023/deploying-remotion-using-the-aws-cdk) was a useful reference for the AWS resource shape; the rest is Terraform-specific.

## What the CLI actually creates

Crack open [`@remotion/lambda`](https://github.com/remotion-dev/remotion/tree/main/packages/lambda) and the deployer is doing four things:

1. Uploading `remotionlambda-arm64.zip` (shipped inside the npm tarball) as the Lambda code.
2. Attaching a region-specific public Chromium layer hosted by Remotion at account `678892195805`.
3. Creating an S3 bucket named `remotionlambda-<id>` for sites and render outputs.
4. Creating an IAM role with permissions for S3, CloudWatch, and `lambda:InvokeFunction` against itself (the orchestrator invokes the renderers).

None of this requires the CLI. You can do all of it in Terraform — you just have to feed it the right inputs.

## Getting the Lambda zip into Terraform

The Lambda zip ships inside the npm tarball. I download and extract it during `terraform plan` so the module is version-pinned by a single variable:

```hcl
locals {
  remotion_tarball_url = "https://registry.npmjs.org/@remotion/lambda/-/lambda-${var.remotion_version}.tgz"
  remotion_zip_path    = "${path.module}/artifacts/remotionlambda-arm64.zip"
}

resource "null_resource" "fetch_remotion_zip" {
  triggers = {
    tarball_url = local.remotion_tarball_url
  }

  provisioner "local-exec" {
    command     = <<-EOT
      mkdir -p artifacts
      curl -sSL ${local.remotion_tarball_url} \
        | tar -xz --strip-components=1 -C artifacts package/remotionlambda-arm64.zip
    EOT
    working_dir = path.module
  }
}

data "local_file" "remotion_zip" {
  filename   = local.remotion_zip_path
  depends_on = [null_resource.fetch_remotion_zip]
}
```

That gives me `artifacts/remotionlambda-arm64.zip` to point `aws_lambda_function.filename` at, with `source_code_hash = data.local_file.remotion_zip.content_base64sha256` so Terraform detects version bumps.

## The Chromium layer ARNs

Remotion publishes one Chromium layer per region. The ARNs and current versions live at `@remotion/lambda/dist/shared/hosted-layers.js` inside the tarball. I extract them into a `layers.tf` map at upgrade time with a small script:

```bash
# scripts/upgrade.sh
tar -xzf remotion-lambda.tgz package/dist/shared/hosted-layers.js
node generate-layers.js
```

`generate-layers.js` reads the `hostedLayers` table Remotion ships — an object keyed by AWS region, each value a list of `{ layerArn, version }` descriptors (Chromium plus any other layers Remotion publishes for that region) — and writes a Terraform map to `layers.tf`:

```js
const { writeFile } = require("node:fs/promises");
const { hostedLayers } = require("./package/dist/shared/hosted-layers.js");

const regions = Object.entries(hostedLayers).map(([region, layers]) => {
  const arns = layers.map((l) => `"${l.layerArn}:${l.version}"`).join(", ");
  return `    "${region}" = [${arns}]`;
});

writeFile("layers.tf", `locals {\n  remotion_layers = {\n${regions.join("\n")}\n  }\n}\n`);
```

The output is a region → layer ARN lookup that the Lambda resource consumes via `layers = local.remotion_layers[var.region]`. Regenerate it whenever you bump `var.remotion_version`. Keeping it auto-generated and committed (rather than reading the file at plan time) keeps the diff visible in PR review.

## The function-name trap

Here's the first thing that will bite you. The Remotion client library — the thing your backend calls to kick off a render — discovers your Lambda by name, not by ARN. The name has to match a fixed pattern ([remotion#5911](https://github.com/remotion-dev/remotion/issues/5911)):

```
remotion-render-{version}-mem{memory}mb-disk{disk}mb-{timeout}sec
```

For example: `remotion-render-4-0-454-mem2048mb-disk2048mb-120sec`.

Two consequences:

- **You cannot apply your house naming convention to this function.** Every other resource in my account follows a `service-env-region` prefix; the Remotion function does not. Document it as an accepted exception or you'll spend an afternoon chasing a 404 from `renderMediaOnLambda()`.
- **A Remotion version bump creates a new Lambda.** Because the version is encoded in the name, bumping `var.remotion_version` doesn't update the existing function — Terraform plans a destroy + create. That's actually fine, and arguably correct (you can roll back to the old function by changing the variable), but it means every upgrade is a swap, not an in-place change.
- **The log group name has to track the function name too**, otherwise CloudWatch will create a second log group implicitly the first time the new function runs.
- **IAM follows the function name as well.** The orchestrator invokes the renderers via `lambda:InvokeFunction` against itself, and the resource ARN in that policy statement contains the function name. Same for `lambda:PutRuntimeManagementConfig` if you use it. If you scope those ARNs (and you should — `Resource = "*"` here is sloppy), the inline policy gets rewritten on every version bump. Make sure your locals derive the policy resource from `local.remotion_function_name` so the two never drift:

```hcl
{
  Sid      = "LambdaSelfInvoke"
  Effect   = "Allow"
  Action   = "lambda:InvokeFunction"
  Resource = "arn:aws:lambda:${var.region}:${data.aws_caller_identity.current.account_id}:function:${local.remotion_function_name}"
}
```

The whole local for the name is just:

```hcl
locals {
  remotion_function_name = "remotion-render-${replace(var.remotion_version, ".", "-")}-mem${var.lambda_memory_size}mb-disk${var.lambda_ephemeral_storage}mb-${var.lambda_timeout}sec"
}
```

…but the implications above are what catch you out, not the string interpolation.

## The 250 MB layer ceiling

Lambda enforces a 250 MB cap on the unzipped size of function code plus all layers. The Remotion Chromium layer is already large — Chromium plus FFmpeg isn't small — and you only have so much headroom left ([remotion#5909](https://github.com/remotion-dev/remotion/issues/5909), [remotion#3839](https://github.com/remotion-dev/remotion/issues/3839)).

This is fine until you want observability. The Datadog Lambda integration ships as two more layers (the extension and the Node runtime), and that combo plus the Remotion layer will push you over the limit. The first time I added Datadog, the apply failed with `CodeStorageExceededException`.

Two things I do to keep this under control:

1. **Pick the smallest viable Datadog layer combination.** The Datadog extension layer is the big one. Only attach the Node layer if you actually want auto-instrumentation; otherwise you can ship traces via the extension alone.
2. **Use the `DataDog/lambda-datadog/aws` module conditionally.** I keep two paths — a vanilla `aws_lambda_function` for the default deploy, and the Datadog module wrapping the same function for environments where I want traces:

```hcl
resource "aws_lambda_function" "remotion_render" {
  count = var.datadog_enabled ? 0 : 1
  # ...same config, no Datadog layers
}

module "remotion_render_datadog" {
  count   = var.datadog_enabled ? 1 : 0
  source  = "DataDog/lambda-datadog/aws"
  version = "4.6.0"

  function_name = local.remotion_function_name
  filename      = "${path.module}/artifacts/remotionlambda-arm64.zip"
  layers        = local.remotion_layers[var.region]

  datadog_extension_layer_version = 94
  datadog_node_layer_version      = 137

  environment_variables = {
    DD_API_KEY_SSM_ARN = "..."
    DD_SITE            = "..."
    DD_SERVICE         = "remotion-lambda"
    DD_VERSION         = var.remotion_version
  }
}
```

If you blow the limit even with the minimal layer set, the escape hatch is to repackage the Chromium layer yourself — Remotion's binaries are open source at [`remotion-dev/lambda-binaries`](https://github.com/remotion-dev/lambda-binaries) — and strip what you don't need. I haven't had to do this yet; tuning the Datadog layer choice has been enough.

## Two pipelines, not one

One last thing the CLI hides: the Lambda function and the Remotion *site* are two different deployments. The function is your video renderer. The site is the static bundle of your React compositions, deployed to the same S3 bucket via `npx remotion lambda sites create`. The Lambda navigates to the site URL each frame.

In a Terraform-managed world that means:

- **Infrastructure pipeline** (this module) owns the Lambda, IAM, bucket, log group. Apply on infra PRs.
- **Application pipeline** (CD on your app repo) owns `remotion lambda sites create`. Deploys on app PRs.

The site name is what ties them together. Pick a deterministic name in the Terraform module, expose it as an SSM parameter or Terraform output, and have CD read it. Don't let `remotion lambda sites create` pick a name for you — you'll end up with mismatches.

## Trade-offs

The downside of all this is real: you've taken on maintenance Remotion's tooling was doing for you. Every version bump is now a checklist — bump the variable, regenerate `layers.tf`, plan, apply, redeploy the site. Datadog layer versions drift independently. The function-name pattern could change in a future Remotion major and silently break your client.

The upside is also real: your video rendering plan/applies the same way as the rest of your AWS account, code review catches changes before they hit prod, and a `terraform destroy` actually cleans everything up. For a team already living in Terraform, it's the right trade.
