# Example: Get digest of container image from Artifact Registry

Recently I played around with the [Cloud Run Terraform resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_run_service) and came accross an issue with redeployment of a servie once the underlying container image has changed. Simply referring to a tag from the container image is not enough. Due to the state based nature of Terraform, Terraform would not pick-up a state change in an external system such as a Container Registry.

When looking around I came accross the following issue [Cloud Run Deployment not updating when Docker image changes](https://github.com/hashicorp/terraform-provider-google/issues/6706).  The recommendation was to use image digest (due to it's unique nature) as can be seen in the [following comment](https://github.com/dlorch/errors.fail/commit/a6b8381f18dae797ae5fd225c390b21eda981f31). 

Since it is a [best practice](https://cloud.google.com/architecture/using-container-images) anyways, I thought it's actually a nice add.

The [approach used](https://github.com/dlorch/errors.fail/commit/a6b8381f18dae797ae5fd225c390b21eda981f31) by another Github user was only for Google Container Registry, and since _Artifact Registry expands on the capabilities of Container Registry and is the recommended container registry for Google Cloud_ [1] I thought it's nice to have the same example adapted for Artifact Registry.

## Implementation

First you need to define the docker provider to connect to the Docker Registry and get the image data.

It's important that the machine you execute Terraform from (e.g. `terraform apply`) has docker cli installed.

```tf
provider "docker" {

  registry_auth {
    address  = "${local.region}-docker.pkg.dev"
    username = "oauth2accesstoken"
    password = data.google_client_config.default.access_token
  }
}
```

We use the token based authentication similar to explained [here](https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling#token).

```tf
provider "docker" {

  registry_auth {
    address  = "${local.region}-docker.pkg.dev"
    username = "oauth2accesstoken"
    password = data.google_client_config.default.access_token
  }
}
```

After we are authenticated against the registry we can define the image as a [data resource](https://www.terraform.io/language/data-sources) (a resource that already exists but we need to access in Terraform).

```tf
data "docker_registry_image" "container_image" {

  name = "${local.region}-docker.pkg.dev/${var.project_id}/${local.service-name}/${local.service-name}"
}
```

Since I use `locals` here the final result would be something like this: 
`us-central1-docker.pkg.dev/jsolarz-argolis-sandbox/cloud-run-hello/cloud-run-hello`

After we have defined the data resource we can access it's [data](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs/data-sources/registry_image).

And that's about it. Now we can use the sha256 digest like `data.docker_registry_image.container_image.sha256_digest` and append it to our images used for Cloud Run.

## References

 - [1] [https://cloud.google.com/artifact-registry/docs/overview#transition](https://cloud.google.com/artifact-registry/docs/overview#transition)