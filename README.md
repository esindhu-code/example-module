# terraform-google-live-regional-lb-https-serverless-negs

This module allows you to create Cloud HTTP(S) Load Balancer with
Serverless Network Endpoint Groups (NEGs) and place serverless services from Cloud Run, Cloud Functions and App Engine behind a Cloud Load Balancer.

## Usage

### With Cloud Run as the Network Endpoint Group
```HCL
resource "google_compute_region_network_endpoint_group" "cloudrun_neg" {
  name                  = "${var.name}-neg"
  network_endpoint_type = "SERVERLESS"
  region                = var.region
  cloud_run {
    service = google_cloud_run_service.run_service.name
  }
}

module "test_regional_lb" {
  source = "./modules/regional-https-lb-serverless-neg"

  # Project and network details
  project_id          = local.project_id
  network_selflink    = data.google_compute_network.rlb_vpc.id
  subnetwork_selflink = data.google_compute_subnetwork.rlb_subnet.self_link 

  # Load Balancer name
  name     = "test-lb"
  location = "us-central1"

  # SSL Certificate details
  private_key = "test-key"
  certificate = <<EOF
-----BEGIN CERTIFICATE-----

-----END CERTIFICATE-----
EOF

  # Target HTTPS Proxy details
  ssl            = true
  min_tls        = "TLS_1_2"
#   policy_profile = "RESTRICTED"

  create_url_map = true

  # Forwarding rule IP address
  create_address = false
  ip_address     = "10.185.7.103"
  port_range     = "443"
  lb_scheme      = "INTERNAL_MANAGED" 

# Host patterns to match
  host = ["vmstest.gcp.farmersinsurance.cloud"]

  backends = {
   default = {
      description = "cloud function"
      path  = ["/voice-to-email/*"]
      groups = {
          group = google_compute_region_network_endpoint_group.cloudfunction_neg.id
          balancing_mode = "UTILIZATION"
          description ="cloud function group"

        }
      iap_config = {
        enable               = false
        oauth2_client_id     = ""
        oauth2_client_secret = ""
      }
      log_config = {
        enable      = true
        sample_rate = null
      }
    }
    run1 ={
        description = "cloud run"
        path  = ["/speech-to-text/*"]
      groups = {
          group = google_compute_region_network_endpoint_group.cloudrun_neg.id          
          balancing_mode = "UTILIZATION"
          description ="cloud run group"
        }
      iap_config = {
        enable               = false
        oauth2_client_id     = ""
        oauth2_client_secret = ""
      }
      log_config = {
        enable      = true
        sample_rate = null
      } 
    }
  }
}

```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| alias\_name | Alias name to which the load balancer needs to connect | `string` | `""` | no |
| backends | Map backend indices to list of backend maps. | <pre>map(object({
    project         = optional(string)
    protocol        = optional(string)
    port_name       = optional(string)
    description     = optional(string)
    enable_cdn      = optional(bool)
    security_policy = optional(string, null)

    connection_draining_timeout_sec = optional(number)
    session_affinity                = optional(string)
    affinity_cookie_ttl_sec         = optional(number)
    locality_lb_policy              = optional(string)
    path                            = list(string)

    log_config = object({
      enable      = optional(bool)
      sample_rate = optional(number)
    })

    groups = object({
      group                        = string
      balancing_mode               = optional(string)
      capacity_scaler              = optional(number)
      description                  = optional(string)
      max_connections              = optional(number)
      max_connections_per_instance = optional(number)
      max_connections_per_endpoint = optional(number)
      max_rate                     = optional(number)
      max_rate_per_instance        = optional(number)
      max_rate_per_endpoint        = optional(number)
      max_utilization              = optional(number)
    })
    iap_config = object({
      enable               = bool
      oauth2_client_id     = optional(string)
      oauth2_client_secret = optional(string)
    })
    cdn_policy = optional(object({
      cache_mode                   = optional(string)
      signed_url_cache_max_age_sec = optional(string)
      default_ttl                  = optional(number)
      max_ttl                      = optional(number)
      client_ttl                   = optional(number)
      negative_caching             = optional(bool)
      negative_caching_policy = optional(object({
        code = optional(number)
        ttl  = optional(number)
      }))
      serve_while_stale = optional(number)
      cache_key_policy = optional(object({
        include_host           = optional(bool)
        include_protocol       = optional(bool)
        include_query_string   = optional(bool)
        query_string_blacklist = optional(list(string))
        query_string_whitelist = optional(list(string))
        include_named_cookies  = optional(list(string))
      }))
    }))
    outlier_detection = optional(object({
      base_ejection_time = optional(object({
        seconds = number
        nanos   = optional(number)
      }))
      consecutive_errors                    = optional(number)
      consecutive_gateway_failure           = optional(number)
      enforcing_consecutive_errors          = optional(number)
      enforcing_consecutive_gateway_failure = optional(number)
      enforcing_success_rate                = optional(number)
      interval = optional(object({
        seconds = number
        nanos   = optional(number)
      }))
      max_ejection_percent        = optional(number)
      success_rate_minimum_hosts  = optional(number)
      success_rate_request_volume = optional(number)
      success_rate_stdev_factor   = optional(number)
    }))
  }))</pre> | n/a | yes |
| balancing\_mode | Specifies the balancing mode for this backend. Possible values are: UTILIZATION, RATE, CONNECTION. | `string` | `"CONNECTION"` | no |
| certificate | The certificate in PEM format. The certificate chain must be no greater than 5 certs long. | `any` | n/a | yes |
| create\_address | Set true to create compute address | `bool` | `false` | no |
| create\_dns\_record | Set true create DNS A record | `bool` | `false` | no |
| create\_url\_map | Set true to create URL map | `bool` | `false` | no |
| dns\_project | The project in which the Private DNS Managed zone is created | `string` | `"gcp-live-core-prod-shd-svcs"` | no |
| dns\_suffix | DNS suffix used to create DNS A record | `string` | `"gcp.farmersinsurance.cloud."` | no |
| dns\_zone | The DNS Zone in which DNS A record is to be created | `string` | `"private-zone-gcp-fi-cloud"` | no |
| host | The list of host patterns to match. | `list(string)` | n/a | yes |
| ip\_address | IP Address to create forwarding rule | `any` | n/a | yes |
| labels | Labels to apply to this forwarding rule. A list of key->value pairs. | `map(any)` | `{}` | no |
| lb\_scheme | Indicates what kind of load balancing this regional backend service will be used for. | `string` | `"INTERNAL"` | no |
| location | Region where the load balancer will be created | `string` | `"us-central1"` | no |
| min\_tls | Minimum version of SSL protocol | `string` | `"TLS_1_0"` | no |
| name | Name of the Load Balancer | `any` | n/a | yes |
| network\_selflink | Self link of the network | `any` | n/a | yes |
| path\_matcher | The name of the PathMatcher to use to match the path portion of the URL if the hostRule matches the URL's host portion. | `string` | `"allpaths"` | no |
| path\_rule | n/a | <pre>list(map(object({
    paths   = list(string)
    service = string
  })))</pre> | `[]` | no |
| policy\_profile | Profile specifies the set of SSL features that can be used by the load balancer when negotiating SSL with clients. | `string` | `"COMPATIBLE"` | no |
| port\_range | Only packets addressed to ports in the specified range will be forwarded to target. If empty, all packets will be forwarded. | `string` | `"443"` | no |
| private\_key | The write-only private key in PEM format | `any` | n/a | yes |
| project\_id | Name of the project where the load balancer will be created | `any` | n/a | yes |
| protocol | The protocol for the backend and frontend forwarding rule. | `string` | `"TCP"` | no |
| security\_policy | The resource URL for the security policy to associate with the backend service | `string` | `null` | no |
| ssl | Set to `true` to enable SSL support, requires variable `ssl_certificates` - a list of self\_link certs | `bool` | `true` | no |
| ssl\_certificates | SSL cert self\_link list. Required if `ssl` is `true` and no `private_key` and `certificate` is provided. | `list(string)` | `[]` | no |
| ssl\_policy | Selfink to SSL Policy | `string` | `null` | yes |
| subnetwork_selflink | Self link of the subnetwork | `string` | n/a | yes |
| url\_map | The url\_map resource to use. Default is to send all traffic to first backend. | `string` | `null` | no |

## Outputs

| Name | Description |
|------|-------------|
| backend\_services | The backend service resources. |
| ip | The external IPv4 assigned to the global fowarding rule. |
| https\_proxy | The HTTPS proxy used by this module. |
| url\_map | The default URL map used by this module. |
