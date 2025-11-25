# MVD (Minimum Viable Dataspace) Integration Guide

This guide explains how to configure the EDC Data Dashboard to work with MVD connectors.

## Prerequisites

- MVD running with connectors (consumer, provider-qna, provider-manufacturing)
- MVD connectors accessible on localhost

## Configuration Steps

### 1. Configure MVD Connectors for CORS

MVD connectors need CORS configuration to allow browser-based dashboard access.

Edit your MVD `docker-compose.yml` and add the following environment variables to each connector service (`consumer-connector`, `provider-connector-qna`, `provider-connector-manufacturing`):

```yaml
environment:
  EDC_WEB_REST_CORS_ENABLED: "true"
  EDC_WEB_REST_CORS_HEADERS: "origin,content-type,accept,authorization,x-api-key"
  EDC_WEB_REST_CORS_ORIGINS: "*"
  EDC_WEB_REST_CORS_METHODS: "GET,PUT,POST,DELETE,OPTIONS"
```

**Important:** Add these lines after the `EDC_RUNTIME_ID` variable in each connector's environment section.

Recreate the containers to apply the changes:

```bash
cd /path/to/MinimumViableDataspace
docker compose up -d consumer-connector provider-connector-qna provider-connector-manufacturing
```

### 2. Configure Dashboard Connector Endpoints

Update `public/config/edc-connector-config.json` with your MVD connector endpoints:

```json
[
  {
    "connectorName": "Consumer",
    "managementUrl": "http://localhost:9081/api/management",
    "defaultUrl": "http://localhost:9080/api",
    "protocolUrl": "http://localhost:9082/api/dsp",
    "federatedCatalogEnabled": false,
    "federatedCatalogUrl": "http://fc-consumer.dataspace/catalog",
    "did": "did:web:consumer-identityhub:7083",
    "apiKey": "password"
  },
  {
    "connectorName": "Provider-qna",
    "managementUrl": "http://localhost:8191/api/management",
    "defaultUrl": "http://localhost:8190/api",
    "protocolUrl": "http://localhost:8192/api/dsp",
    "federatedCatalogEnabled": false,
    "federatedCatalogUrl": "http://fc-provider.dataspace/catalog",
    "did": "did:web:provider-identityhub:7093",
    "apiKey": "password"
  },
  {
    "connectorName": "Provider-manufacturing",
    "managementUrl": "http://localhost:8291/api/management",
    "defaultUrl": "http://localhost:8290/api",
    "protocolUrl": "http://localhost:8292/api/dsp",
    "federatedCatalogEnabled": false,
    "federatedCatalogUrl": "http://fc-provider.dataspace/catalog",
    "did": "did:web:provider-identityhub:7093",
    "apiKey": "password"
  }
]
```

**Key configuration points:**

- `managementUrl`: Must include `/api/management` path prefix
- `defaultUrl`: Used for health checks and observability endpoints
- `protocolUrl`: DSP protocol endpoint with `/api/dsp` path
- `apiKey`: MVD default API key is `password`

### 3. Run the Dashboard

Using Docker:

```bash
docker build -t eclipse-edc/data-dashboard .
docker run -p 8082:8080 \
  -v $PWD/public/config/:/app/config \
  -v $PWD/nginx.conf:/etc/nginx/conf.d/default.conf \
  eclipse-edc/data-dashboard
```

Access the dashboard at http://localhost:8082

### 4. Verify Connection

1. Open http://localhost:8082 in your browser
2. Select a connector from the dropdown
3. Verify the connection status shows "Connected" (green indicator)
4. Check browser DevTools Console for any CORS errors

## Troubleshooting

### Connection Failed

**Symptoms:** Dashboard shows "Not Connected" or "Connection Failed"

**Solutions:**

1. Verify CORS is enabled in MVD connectors:
   ```bash
   docker exec mvd-consumer-connector env | grep CORS
   ```
   
2. Check connector health endpoint:
   ```bash
   curl http://localhost:9080/api/check/health
   ```

3. Test CORS headers:
   ```bash
   curl -v -X OPTIONS http://localhost:9080/api/check/health \
     -H "Origin: http://localhost:8082" \
     -H "Access-Control-Request-Method: GET"
   ```
   
   Should return `Access-Control-Allow-Origin: *` header

4. Verify Management API access with authentication:
   ```bash
   curl -X POST http://localhost:9081/api/management/v3/assets/request \
     -H "Content-Type: application/json" \
     -H "X-Api-Key: password" \
     -d '{"@context": {"@vocab": "https://w3id.org/edc/v0.0.1/ns/"}, "@type": "QuerySpec"}'
   ```

### CORS Errors in Browser Console

If you see CORS errors in the browser console, ensure:

1. MVD connectors have been recreated (not just restarted) after adding CORS configuration
2. The CORS environment variables are correctly set in docker-compose.yml
3. All connectors (consumer, provider-qna, provider-manufacturing) have the CORS configuration

## Port Mapping

| Connector | Default API | Management API | Protocol API | Data Plane |
|-----------|-------------|----------------|--------------|------------|
| Consumer | 9080 | 9081 | 9082 | 11001 |
| Provider QnA | 8190 | 8191 | 8192 | 12001 |
| Provider Manufacturing | 8290 | 8291 | 8292 | 12002 |

## Security Notes

⚠️ **Warning:** The configuration in this guide uses:
- CORS with `origins: "*"` - allows any origin (suitable for development only)
- Default API key `password` - should be changed in production
- Connector configuration stored in browser local storage when `enableUserConfig: true`

For production deployments:
- Restrict CORS origins to specific domains
- Use strong API keys and rotate them regularly
- Consider using proper authentication/authorization mechanisms
- Do not commit sensitive configuration to version control
