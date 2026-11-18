# ---------- Data sources: Secret Manager ----------

data "google_secret_manager_secret_version" "oidc_client_secret" {
  secret = "oidc-client-secret" # имя секрета в Secret Manager
  # version_id = "latest" # по умолчанию берётся latest
}

# ---------- Workforce pool ----------

resource "google_iam_workforce_pool" "pool" {
  workforce_pool_id = var.workforce_pool_id
  parent            = "organizations/${var.org_id}"
  location          = "global"

  display_name = "Corporate workforce pool"
  description  = "External IdP federation for workforce identities"
  session_duration = "3600s"
}

# ---------- Provider ----------

resource "google_iam_workforce_pool_provider" "provider" {
  location          = "global"
  workforce_pool_id = google_iam_workforce_pool.pool.workforce_pool_id
  provider_id       = var.provider_id

  display_name = "Entra ID provider"
  description  = "OIDC federation via Microsoft Entra ID"

  attribute_condition = "assertion['groups'].exists(g, g == 'gcp-users')"

  attribute_mapping = {
    "google.subject"        = "assertion.sub"
    "google.display_name"   = "assertion.name"
    "google.groups"         = "assertion.groups"
    "google.email"          = "assertion.email"
    "google.profile_image"  = "assertion.picture"
  }

  oidc {
    issuer_uri        = var.oidc_issuer_uri
    client_id         = var.oidc_client_id
    client_secret {
      value {
        secret_manager = data.google_secret_manager_secret_version.oidc_client_secret.secret
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

  display_name = "Corp SCIM tenant"
  description  = "SCIM sync of users/groups from the corporate IdP"

  claim_mapping = {
    "google.subject" = "user.externalId"
    "google.group"   = "group.externalId"
  }

  hard_delete       = false
  deletion_policy   = "PREVENT"

  depends_on = [google_iam_workforce_pool_provider.provider]
}

# ---------- IAM bindings for SCIM service agent ----------

# Получаем service_agent из созданного SCIM tenant (output-only поле)
data "google_iam_workforce_pool_provider_scim_tenant" "current" {
  name = google_iam_workforce_pool_provider_scim_tenant.scim_tenant.name
}

resource "google_project_iam_member" "scim_service_agent_roles" {
  project = var.project_id # обязательно укажи свой project_id в переменных

  # Роль, которая позволяет создавать/обновлять пользователей и группы в workforce pool
  role = "roles/iam.workforcePoolAdmin"
  member = "serviceAccount:${data.google_iam_workforce_pool_provider_scim_tenant.current.service_agent.email}"
}
