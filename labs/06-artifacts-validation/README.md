# Validating Artifacts and Deployments

## Learning Goals

* Understanding artifact integrity and provenance
* Signing and validating Spring Boot artifacts with Sigstore Cosign
* Attesting and validating the provenance of Spring Boot artifacts with SLSA

## Overview

In this exercise, you will ensure the integrity of your Spring Boot artifacts through signatures, attestations, and validation.

## Details

### Signing Spring Boot OCI Images with Cosign

Let's start by containerizing the `integrity-gradle-demo` application.

```shell
cd integrity-gradle-demo
./gradlew bootBuildImage
```

Then, tag the image and publish it to your GitHub Container Registry. You might need to authenticate with the registry (`docker login ghcr.io`). Follow these [instructions](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-to-the-container-registry) to generate a Personal Access Token from your GitHub account.

```shell
docker tag integrity-gradle-demo ghcr.io/<your-github-username>/integrity-gradle-demo
docker push ghcr.io/<your-github-username>/integrity-gradle-demo
```

To validate the authenticity of the image before deploying it, you can apply a cryptographic signature using [Sigstore Cosign](https://docs.sigstore.dev/signing/quickstart/).

Ensure you have Cosign installed (`cosign version`) or follow the [instructions](https://docs.sigstore.dev/signing/quickstart/#installation) to install it.

Using Cosign, you can sign the OCI Image, publish the signature to the same container registry as the image, and attach it to the image. A keyless approach will be used for authentication, similar to what you experienced in lab 1 when signing commits.

```shell
cosign sign --yes ghcr.io/<your-github-username>/integrity-gradle-demo
```

Consumers of this image can now validate the signature before deploying it, making sure they are using the image built by you and not one that has been tampered with. One way of doing so is through the Cosign CLI.

```shell
cosign verify \
   --certificate-identity-regexp <your-github-email-address> \
   --certificate-oidc-issuer https://github.com/login/oauth \
   ghcr.io/<your-github-username>/integrity-gradle-demo | jq
```

You can see all the supply chain security related artifacts for an image using the Cosign CLI.

```shell
cosign tree ghcr.io/<your-github-username>/integrity-gradle-demo
```

Feel free to delete the artifacts pushed to `ghcr.io/<your-github-username>/integrity-gradle-demo` from your GitHub account.

### Validating SLSA Build Attestations

Modern build platforms like GitHub and GitLab support attesting the provenance of your Spring Boot builds based on the [SLSA](https://slsa.dev) Framework, in particular adhering to the [Level 3 Build](https://slsa.dev/spec/v1.0/levels#build-l3) criteria for hardened builds.

Given the image `ghcr.io/thomasvitale/band-service`, ensure it is safe to deploy.

First, get a hold of all the supply chain security related artifacts attached to this image. In a real scenario, you want to validate images based on a specific digest. For this example, we'll use the implicit latest tag. Don't do this for production artifacts!

```shell
cosign tree ghcr.io/thomasvitale/band-service
```

You can see there are two main artifacts: a signature and a SLSA provenance attestation.

First, verify the signature is legit. This image was built from the repository https://github.com/ThomasVitale/band-service as a GitHub Actions workflow, relying on the keyless authentication support provided by GitHub.

```shell
cosign verify \
   --certificate-identity-regexp https://github.com/ThomasVitale \
   --certificate-oidc-issuer https://token.actions.githubusercontent.com \
   ghcr.io/thomasvitale/band-service | jq
```

Next, validate the SLSA Provenance attestation attached to the image. This attestation includes all the main information about how, when, and where the image was built. It's a critical piece of information to assess whether you want to trust a specific artifact to run in your environments.

```shell
cosign verify-attestation --type slsaprovenance \
   --certificate-identity-regexp https://github.com/slsa-framework \
   --certificate-oidc-issuer https://token.actions.githubusercontent.com \
   ghcr.io/thomasvitale/band-service | jq .payload -r | base64 --decode | jq
```

Feel free to analyze the result to get insights into the image provenance.

The SLSA framework publishes also a dedicated [SLSA Verifier](https://github.com/slsa-framework/slsa-verifier) CLI you can use to validate SLSA Attestations.
