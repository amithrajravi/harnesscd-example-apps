# Troubleshooting Connector Reference Issue

## Error: "Could not fetch manifests. Connectors with identifier(s) [ownappgitconnector] not found"

### Quick Fixes to Try:

1. **Verify Connector Identifier in Harness UI:**
   - Go to: Connectors → Find your GitHub connector
   - Check the exact identifier (case-sensitive, no typos)
   - Make sure it matches: `ownapp_gitconnector`

2. **Re-import/Update the Connector:**
   - In Harness UI, go to Connectors
   - Either create new or edit existing connector
   - Use the YAML from `github-connector.yml`
   - Make sure to set scope to "Project" (default_project)
   - Save the connector

3. **Test the Connector:**
   - In the connector details page, click "Test" 
   - Verify it connects successfully to GitHub
   - If test fails, check your GitHub PAT secret (`ownappgitpat`)

4. **Verify Service YAML:**
   - Make sure `service.yml` line 15 has: `connectorRef: ownapp_gitconnector`
   - Re-import the service YAML in Harness UI
   - Or edit the service manually and select the connector from dropdown

5. **Check Connector Scope:**
   - The connector must be in the same project (`default_project`)
   - Same organization (`default`)
   - Same account (`exqRN-uBSkKCxM4tS2ZLBg`)

### Alternative: Use Account-Level Connector Reference

If the connector is created at Account level, reference it as:
```yaml
connectorRef: account.ownapp_gitconnector
```

If connector is at Org level:
```yaml
connectorRef: org.ownapp_gitconnector
```

If connector is at Project level (current setup):
```yaml
connectorRef: ownapp_gitconnector
```

### Common Issues:

- **Secret not found**: Ensure secret `ownappgitpat` exists in Harness Secrets
- **Wrong identifier**: Check for typos or case differences
- **Connector not saved**: Make sure connector was saved successfully in UI
- **Branch name**: Ensure the branch is `master` or ` through the UI to confirm the exact branch name

