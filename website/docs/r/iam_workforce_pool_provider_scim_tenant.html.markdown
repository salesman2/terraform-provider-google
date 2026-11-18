# ---------- Variables ----------

variable "org_id" {
  type        = string
  description = "Organization ID (numeric)"
}

variable "project_id" {
  type        = string
  description = "Project ID for Secret Manager access"
}

variable "workforce_pool_id" {
  type        = string
  validation {
    condition     = can(regex("^[a-z0-9-]{4,32}$", var.workforce_pool_id))
    error_message = "4-32 chars: a-z, 0-9, dash."
  }
}

variable "provider_id" {
  type = string
}

variable "scim_tenant_id" {
  type        = string
  validation {
    condition     = can(regex("^[a-z0-9-]{4,32}$", var.scim_tenant_id))
    error_message = "4-32 chars: a-z, 0-9, dash."
  }
}

variable "oidc_issuer_uri" {
  type = string
}

variable "oidc_client_id" {
  type = string
}

variable "oidc_client_secret_secret_name" {
  type        = string
  description = "Name of the secret in Secret Manager (not the value!)"
  default     = "oidc-client-secret"
}

# ---------- Data: Secret Manager ----------

data "google_secret_manager_secret_version" "oidc_client_secret" {
  project    = var.project_id
  secret     = var.oidc_client_secret_secret_name
  version_id = "latest"
}

# ---------- Workforce pool ----------

resource "google_iam_workforce_pool" "pool" {
  workforce_pool_id = var.workforce_pool_id
  parent            = "organizations/${var.org_id}"
  location          = "global"

  display_name     = "Corporate workforce pool"
  description      = "External IdP federation for workforce identities"
  session_duration = "3600s"
}

# ---------- Provider (OIDC) ----------

resource "google_iam_workforce_pool_provider" "provider" {
  location          = "global"
  workforce_pool_id = google_iam_workforce_pool.pool.workforce_pool_id
  provider_id       = var.provider_id

  display_name = "Entra ID provider"
  description  = "OIDC federation via Microsoft Entra ID"

  # Гейт на конкретную группу — не оставляем "true" в проде
  attribute_condition = "assertion['groups'].exists(g, g == 'gcp-users')"

  attribute_mapping = {
    "google.subject"      = "assertion.sub"
    "google.display_name" = "assertion.name"
    "google.groups"       = "assertion.groups"
    "google.email"        = "assertion.email"
  }

  oidc {
    issuer_uri = var.oidc_issuer_uri
    client_id  = var.oidc_client_id

    client_secret {
      value {
        # secret_manager ждёт имя секрета (projects/.../secrets/...), не значение
        secret_manager = data.google_secret_manager_secret_version.oidc_client_secret.id
      }
    }

    web_sso_config {
      response_type             = "CODE"
      assertion_claims_behavior = "MERGE_USER_INFO_OVER_ID_TOKEN_CLAIMS"
      additional_scopes         = ["groups", "roles"]
    }
  }
}

# ---------- SCIM tenant ----------

resource "google_iam_workforce_pool_provider_scim_tenant" "scim_tenant" {
  location          = "global"
  workforce_pool_id = google_iam_workforce_pool.pool.workforce_pool_id
  provider_id       = google_iam_workforce_pool_provider.provider.provider_id
  scim_tenant_id    = var.scim_tenant_id

  display_name = "Corp SCIM tenant"   # максимум 32 символа
  description  = "SCIM sync of users/groups from the corporate IdP"

  claim_mapping = {
    "google.subject" = "user.externalId"
    "google.group"   = "group.externalId"
  }

  hard_delete     = false     # soft-delete даёт ~30 дней на откат
  deletion_policy = "PREVENT" # защита от случайного terraform destroy

  depends_on = [google_iam_workforce_pool_provider.provider]

  # update-таймаут по умолчанию 0 минут — любая правка упадёт мгновенно
  timeouts {
    create = "20m"
    update = "20m"
    delete = "20m"
  }
}

# ---------- IAM для SCIM service agent ----------

# service_agent приходит в виде principal, напр. "serviceAccount:...@gcp-sa-scim.iam.gserviceaccount.com"
locals {
  scim_agent_member = (
    startswith(google_iam_workforce_pool_provider_scim_tenant.scim_tenant.service_agent, "serviceAccount:")
    ? google_iam_workforce_pool_provider_scim_tenant.scim_tenant.service_agent
    : "serviceAccount:${google_iam_workforce_pool_provider_scim_tenant.scim_tenant.service_agent}"
  )
}

# Роль выдаём на уровне пула, а не проекта — минимально достаточный scope
resource "google_iam_workforce_pool_iam_member" "scim_agent_pool_admin" {
  workforce_pool_name = google_iam_workforce_pool.pool.name
  role                = "roles/iam.workforcePoolAdmin"
  member              = local.scim_agent_member

  depends_on = [google_iam_workforce_pool_provider_scim_tenant.scim_tenant]
}

# ---------- Outputs ----------

output "scim_tenant_name" {
  value = google_iam_workforce_pool_provider_scim_tenant.scim_tenant.name
}

output "scim_tenant_base_uri" {
  value = google_iam_workforce_pool_provider_scim_tenant.scim_tenant.base_uri
}

output "scim_tenant_state" {
  value = google_iam_workforce_pool_provider_scim_tenant.scim_tenant.state
}

output "scim_service_agent" {
  value = google_iam_workforce_pool_provider_scim_tenant.scim_tenant.service_agent
}
