---
title: API Reference
last_updated: YYYY-MM-DD
status: template
---

# API Reference

## Authentication

<!-- Describe auth mechanism: API key, JWT, OAuth, etc. -->

## Base URLs

| Environment | URL |
|-------------|-----|
| Development | `http://localhost:PORT` |
| Staging | {{STAGING_URL}} |
| Production | {{PRODUCTION_URL}} |

## Endpoints

### Group Name

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| <!-- GET/POST --> | <!-- /api/... --> | <!-- What it does --> | <!-- Required? --> |

## Error Handling

| Status Code | Meaning | When |
|-------------|---------|------|
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Missing/invalid auth |
| 404 | Not Found | Resource doesn't exist |
| 500 | Internal Error | Server-side failure |
