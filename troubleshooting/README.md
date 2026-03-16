# Troubleshooting

[To Home](../README.md)

## Häufige Probleme & Lösungen bei der Docker Instanz

### Port 5678 ist bereits belegt

```Code
Error: bind: address already in use
```

**Lösung:** Ändere in der `docker-compose.yml` den linken Port, z. B. auf `5679`:

```yaml
ports:
  - "5679:5678"
```

Dann erreichst du n8n unter `http://localhost:5679`.

---

### n8n startet nicht / Container crasht

Prüfe die Logs im Terminal:

```bash
docker compose logs n8n
```

---

Last updated: March 2026
