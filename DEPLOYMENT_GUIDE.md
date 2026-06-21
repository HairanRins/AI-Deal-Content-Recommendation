# Guide de Déploiement — AI Deal Content Recommendation

## Prérequis

### Comptes & Credentials

| Service      | Credential ID             | Permissions requises                               |
| ------------ | ------------------------- | -------------------------------------------------- |
| Zoho CRM     | `zoho-oauth-prod`         | Deals (read), Contacts (read), Accounts (read)     |
| OpenAI       | `openai-prod-api`         | GPT-4o access, JSON mode                           |
| Gmail        | `gmail-oauth-prod`        | Send emails, read contacts                         |
| Slack        | `slack-prod-bot`          | Post to `#workflow-alerts`, `#deal-content-review` |
| Webhook Auth | `webhook-auth-credential` | Header token pour sécuriser le webhook             |

### Variables d'Environnement (n8n)

```bash
# APIs de contenu
CASE_STUDIES_API_URL=https://api.yourcms.com/case-studies
WHITEPAPERS_API_URL=https://api.yourcms.com/whitepapers
FALLBACK_CONTENT_URL=https://api.yourcms.com/content-fallback

# Logging & Monitoring
LOG_API_URL=https://api.yoursite.com/workflow-logs
ERROR_LOG_API_URL=https://api.yoursite.com/error-logs

# Alertes
ALERT_EMAIL=ops@yoursite.com

# Environnement
NODE_ENV=production
```

---

## Configuration Zoho CRM

### 1. Créer le Workflow Rule

```
Module: Deals
Trigger: On Stage Change
Condition: Stage = "Needs Analysis" OR Stage = "Proposal/Price Quote"
Action: Webhook
  URL: https://your-n8n-instance.com/webhook/deal-stage-webhook-v2-prod
  Method: POST
  Headers:
    X-API-Key: [votre-webhook-auth-token]
  Body:
    {
      "dealId": "${Deals.Deal_ID}",
      "stage": "${Deals.Stage}",
      "correlationId": "${Deals.Deal_ID}-${NOW()}"
    }
```

### 2. Mapper les champs requis dans Zoho

Assurez-vous que ces champs sont remplis dans vos deals :

- `Deal_Name` (obligatoire)
- `Stage` (obligatoire)
- `Amount` (optionnel, pour le human gate)
- `Description` (optionnel, pour NLP)
- `Contact_Name` → avec email
- `Account_Name` → avec industry et employees
- `Lead_Source` (optionnel)

---

## Configuration OpenAI

### Model Pinning

Le workflow utilise `gpt-4o-2024-08-06` (version pinned). **Ne changez pas** sans tester.

### Prompt Tuning

Le system prompt est dans le nœud `Prepare AI Prompt`. Pour ajuster :

1. Modifier le prompt dans le nœud
2. Tester avec 5-10 deals réels
3. Vérifier la qualité des recommendations
4. Ajuster les critères de sélection

### Coût estimé

- ~2000 tokens par exécution
- ~$0.01 USD par deal (GPT-4o)
- Budget mensuel: $30-50 pour 3000-5000 deals

---

## Configuration Email

### Templates dynamiques

Le workflow génère l'email via IA. Pour ajouter un template statique fallback :

1. Modifier le nœud `Build AI Recommendation Payload`
2. Ajouter un champ `emailTemplateOverride`
3. Conditionner l'IA ou le template selon le flag

### DKIM/SPF

Assurez-vous que votre domaine a les enregistrements DKIM/SPF configurés pour Gmail.

---

## Monitoring & Alertes

### Dashboards recommandés

#### Métriques clés à tracker

| Métrique                    | Source         | Seuil d'alerte |
| --------------------------- | -------------- | -------------- |
| Taux d'erreur               | Execution logs | > 5%           |
| Temps d'exécution moyen     | n8n Insights   | > 30s          |
| Usage fallback content      | Log API        | > 10%          |
| Score de confiance IA moyen | Log API        | < 0.7          |
| Deals sans email envoyé     | Log API        | > 5%           |

### Alertes Slack

- `#workflow-alerts` : Toutes les erreurs
- `#critical-alerts` : Erreurs critiques (payment, auth, timeout)
- `#deal-content-review` : Review humaine requise (deals > $50k)

---

## Testing

### Test d'intégration

```bash
curl -X POST https://your-n8n-instance.com/webhook/deal-stage-webhook-v2-prod   -H "X-API-Key: votre-token"   -H "Content-Type: application/json"   -d '{
    "dealId": "TEST-DEAL-001",
    "stage": "Needs Analysis",
    "correlationId": "test-001"
  }'
```

### Scénarios de test

1. **Happy path** : Deal standard, APIs up, email envoyé
2. **API down** : Simuler 500 sur case studies API → vérifier fallback
3. **Deal high-value** : Amount = $100k → vérifier review queue Slack
4. **Invalid input** : Missing dealId → vérifier validation error
5. **AI format error** : Tester avec contenu bizarre → vérifier validation

---

### Rotation des credentials

- Zoho OAuth: tous les 90 jours
- OpenAI API key: tous les 180 jours
- Gmail OAuth: tous les 90 jours

---

## 🆘 Troubleshooting

### "No JSON found in AI output"

→ Vérifier que le model supporte `response_format: json_object`. Si non, revenir au parsing regex.

### "API timeout"

→ Augmenter le timeout HTTP Request à 15000ms. Vérifier la latence de votre CMS.

### "Email not sent"

→ Vérifier que `contactEmail` est bien mappé dans Zoho. Vérifier les quotas Gmail.

### "Fallback content always used"

→ Vérifier les URLs des APIs dans les variables d'environnement. Tester les endpoints directement.
