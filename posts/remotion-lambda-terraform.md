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

[Remotion](https://www.remotion.dev/) is a React-based video framework. Its `@remotion/lambda` package distributes rendering across AWS: one orchestrator Lambda fans out frames to up to 200 renderer Lambdas, then stitches the result back together. The official deployment tool is a CLI (`npx remotion lambda functions deploy`) that creates a Lambda function, an S3 bucket, an IAM role, and a log group on your behalf.

The CLI works well for a greenfield project. It fits badly if you already manage AWS through Terraform and want video rendering in the same plan/apply loop as the rest of your infrastructure.

This post shows how I rebuilt the Remotion Lambda setup as a Terraform module, and covers the two non-obvious problems that hit me in production: the hard-coded function name during version upgrades, and the 250 MB layer ceiling once you add Datadog. I assume you're comfortable with Terraform and Lambda. Benjamin Kunkel's [CDK writeup](https://bndkt.com/blog/2023/deploying-remotion-using-the-aws-cdk) was a useful reference for the AWS resource shape; the rest is Terraform-specific.

## What the CLI actually creates

The deployer in [`@remotion/lambda`](https://github.com/remotion-dev/remotion/tree/main/packages/lambda) does four things:

1. Uploads `remotionlambda-arm64.zip` (shipped inside the npm tarball) as the Lambda code.
2. Attaches a region-specific public Chromium layer that Remotion hosts in account `678892195805`.
3. Creates an S3 bucket named `remotionlambda-<id>` for sites and render outputs.
4. Creates an IAM role that allows S3 access, CloudWatch logging, and `lambda:InvokeFunction` against itself, because the orchestrator invokes the renderers.

None of these steps requires the CLI. Terraform can create all four resources; you just have to feed it the right inputs.

## Getting the Lambda zip into Terraform

The Lambda zip ships inside the npm tarball. I download and extract it during `terraform plan`, so a single variable pins the module to a Remotion version:

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

This produces `artifacts/remotionlambda-arm64.zip` for `aws_lambda_function.filename` to point at. Setting `source_code_hash = data.local_file.remotion_zip.content_base64sha256` lets Terraform detect version bumps.

## The Chromium layer ARNs

Remotion publishes one Chromium layer per region. The ARNs and current versions live in `@remotion/lambda/dist/shared/hosted-layers.js` inside the tarball. At upgrade time, a small script extracts them into a `layers.tf` map:

```bash
# scripts/upgrade.sh
tar -xzf remotion-lambda.tgz package/dist/shared/hosted-layers.js
node generate-layers.js
```

`generate-layers.js` reads the `hostedLayers` table that Remotion ships. The table is an object keyed by AWS region; each value lists `{ layerArn, version }` descriptors for Chromium and any other layers Remotion publishes in that region. The script writes the table as a Terraform map in `layers.tf`:

```js
const { writeFile } = require("node:fs/promises");
const { hostedLayers } = require("./package/dist/shared/hosted-layers.js");

const regions = Object.entries(hostedLayers).map(([region, layers]) => {
  const arns = layers.map((l) => `"${l.layerArn}:${l.version}"`).join(", ");
  return `    "${region}" = [${arns}]`;
});

writeFile("layers.tf", `locals {\n  remotion_layers = {\n${regions.join("\n")}\n  }\n}\n`);
```

The Lambda resource consumes the map through `layers = local.remotion_layers[var.region]`. Regenerate the map whenever you bump `var.remotion_version`. I commit the generated `layers.tf` rather than parsing `hosted-layers.js` at plan time, so every layer change shows up as a reviewable diff in the PR.

## The function-name trap

The Remotion client library — the code your backend calls to start a render — discovers your Lambda by name, not by ARN. The name must match a fixed pattern ([remotion#5911](https://github.com/remotion-dev/remotion/issues/5911)):

```
remotion-render-{version}-mem{memory}mb-disk{disk}mb-{timeout}sec
```

For example: `remotion-render-4-0-454-mem2048mb-disk2048mb-120sec`.

The Terraform local for the name is a single interpolation:

```hcl
locals {
  remotion_function_name = "remotion-render-${replace(var.remotion_version, ".", "-")}-mem${var.lambda_memory_size}mb-disk${var.lambda_ephemeral_storage}mb-${var.lambda_timeout}sec"
}
```

The interpolation is trivial; the four consequences of the pattern are not:

- **Your naming convention doesn't apply.** Every other resource in my account follows a `service-env-region` prefix; the Remotion function can't. Document the exception, or you'll spend an afternoon chasing a 404 from `renderMediaOnLambda()`.
- **A version bump replaces the function.** The version is encoded in the name, so bumping `var.remotion_version` makes Terraform plan a destroy and a create, not an in-place update. That plan is acceptable — you can roll back by reverting the variable — but it means every upgrade is a swap.
- **The log group name must track the function name.** Otherwise CloudWatch creates a second log group implicitly the first time the new function runs.
- **The IAM policy must track the function name.** The orchestrator invokes the renderers through `lambda:InvokeFunction` on itself, so the resource ARN in that policy statement contains the function name. The same applies to `lambda:PutRuntimeManagementConfig` if you use it.

Scope those IAM ARNs — `Resource = "*"` is sloppy here — and accept that the inline policy changes on every version bump. Derive the policy resource from `local.remotion_function_name` so the policy and the function never drift:

```hcl
{
  Sid      = "LambdaSelfInvoke"
  Effect   = "Allow"
  Action   = "lambda:InvokeFunction"
  Resource = "arn:aws:lambda:${var.region}:${data.aws_caller_identity.current.account_id}:function:${local.remotion_function_name}"
}
```

## The 250 MB layer ceiling

Lambda caps the unzipped size of a function's code plus all of its layers at 250 MB (262,144,000 bytes). Remotion's own function code plus its Chromium layer leave almost no headroom: a 4.0.431 deploy with the Apple-emoji runtime preference exceeded the cap by about 6 MB with no third-party layers attached ([remotion#6745](https://github.com/remotion-dev/remotion/issues/6745)).

The headroom matters as soon as you want observability. The Datadog Lambda integration ships as two more layers, the extension and the Node runtime. Adding both on top of the Remotion layer pushed my deploy over the cap, and the apply failed with `InvalidParameterValueException: Function code combined with layers exceeds the maximum allowed size of 262144000 bytes`.

I do two things to keep the total under the limit:

1. **Pick the smallest viable Datadog layer set.** The extension layer is the one you can't drop: it forwards telemetry to Datadog, and on its own it ships logs and metrics. Attach the Node runtime layer only if you want automatic APM instrumentation.
2. **Wrap the function in the `DataDog/lambda-datadog/aws` module only where you need it.** I keep two paths: a vanilla `aws_lambda_function` for the default deploy, and the Datadog module wrapping the same configuration for environments where I want telemetry:

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

If you exceed the limit even with the minimal layer set, one option remains: repackage the Chromium layer yourself and strip what you don't need. Remotion's binaries are open source at [`remotion-dev/lambda-binaries`](https://github.com/remotion-dev/lambda-binaries). I haven't needed this option; choosing the Datadog layers carefully has been enough.

## Two pipelines, not one

The CLI hides one more split: the Lambda function and the Remotion *site* are two separate deployments. The function renders the video. The site is the static bundle of your React compositions, and `npx remotion lambda sites create` deploys it to the same S3 bucket. The function opens the site URL to render each frame.

In a Terraform-managed setup, the split means two pipelines:

- **The infrastructure pipeline** (this module) owns the Lambda function, IAM role, S3 bucket, and log group. It applies on infrastructure PRs.
- **The application pipeline** (CD in your app repo) runs `remotion lambda sites create`. It deploys on application PRs.

The site name ties the two pipelines together. Pick a deterministic name in the Terraform module, expose it as an SSM parameter or a Terraform output, and have CD read it. Don't let `remotion lambda sites create` generate a name for you — the two pipelines will drift into mismatched names.

## Trade-offs

The downside is real: you now maintain what Remotion's tooling maintained for you. Every version bump becomes a checklist — bump the variable, regenerate `layers.tf`, plan, apply, redeploy the site. Datadog layer versions drift independently. A future Remotion major release could change the function-name pattern and silently break your client.

The upside is real too. Video rendering plans and applies like the rest of your AWS account. Code review catches changes before they reach production. A `terraform destroy` actually cleans everything up. For a team already living in Terraform, it's the right trade.
