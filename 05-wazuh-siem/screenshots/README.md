# Wazuh SIEM Screenshots

This directory contains supporting evidence from the Wazuh SIEM and detection-engineering portion of the Enterprise Cybersecurity Home Lab.

The screenshots demonstrate:

- Successful Wazuh agent deployment
- Windows Security Event collection
- Sysmon telemetry
- Custom detection-rule validation
- Windows command-line monitoring
- Security-event investigation

## Evidence



| Screenshot | Description |
|---|---|
| <img width="2048" height="1044" alt="Image Oct 20, 2026, 02_12_16 PM" src="https://github.com/user-attachments/assets/b766657a-9f4f-43f1-b701-3ff85da569e2" /> | Wazuh dashboard showing monitored Windows agents online |
| <img width="1920" height="836" alt="679378350-e9753edf-19bb-4932-9788-9dfd53b96268" src="https://github.com/user-attachments/assets/8d008557-e023-451f-8cab-469d01f52120" /> | Windows process creation telemetry from Client01 |
| <img width="1920" height="836" alt="679378350-e9753edf-19bb-4932-9788-9dfd53b96268" src="https://github.com/user-attachments/assets/043044b9-1480-4e58-974c-baaa94f31767" /> | Sysmon process telemetry from DC1 |
| <img width="1920" height="836" alt="679378350-e9753edf-19bb-4932-9788-9dfd53b96268" src="https://github.com/user-attachments/assets/9d77aaef-66d9-4e9e-a2d7-65bbfdb79767" /> | Successful custom Wazuh Rule 100103 detection |
| <img width="1915" height="844" alt="Image Oct 20, 2026, 02_50_45 PM" src="https://github.com/user-attachments/assets/5967ec88-4701-403d-b74f-22542d8e6a20" /> | Successful custom Wazuh Rule 100104 detection |
| <img width="1912" height="852" alt="Screenshot From 2026-10-20 14-59-36" src="https://github.com/user-attachments/assets/aae6cbb3-ed0c-43c3-a2d8-8565687e7008" /> | Overall Wazuh monitoring/dashboard view |

## Security Note

Screenshots are reviewed before publication to ensure they do not expose:

```text
Passwords
Authentication Tokens
Private Keys
VPN Credentials
Sensitive Personal Information
```

The screenshots are intended to demonstrate technical implementation and detection validation without exposing sensitive credentials.

---

[← Detection Testing](../detection-testing.md)

[← Custom Detection Rules](../custom-detection-rules.md)

[← Wazuh SIEM Deployment](../wazuh-deployment.md)
