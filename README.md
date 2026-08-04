# CCIE Security v6.1 学習メモ

このリポジトリは **Cisco CCIE Security v6.1 Blueprint** に基づき、  
学習内容・設定例・検証ログ・図解を体系的にまとめたものです。

GitHub Pages によるドキュメントサイトとして閲覧できるように  
`docs/` 以下に Blueprint 構成を再現しています。

---

# 📚 Blueprint Index（自動リンク）

## 1.0 Perimeter Security and Intrusion Prevention
- [1.1 Deployment modes on ASA/FTD](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.1-Deployment-modes-on-ASA-FTD/README.md)
- [1.2 Firewall features on ASA/FTD](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.2-Firewall-features-on-ASA-FTD/README.md)
- [1.3 Security features on IOS](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.3-Security-features-on-IOS/README.md)
- [1.4 FMC features](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.4-FMC-features/README.md)
- [1.5 NGIPS deployment modes](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.5-NGIPS-deployment-modes/README.md)
- [1.6 NGFW features](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.6-NGFW-features/README.md)
- [1.7 Attack detection](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.7-Attack-detection/README.md)
- [1.8 Clustering and HA](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.8-Clustering-and-HA/README.md)
- [1.9 Policies and rules](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.9-Policies-and-rules/README.md)
- [1.10 Routing protocol security](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.10-Routing-protocols-security/README.md)
- [1.11 Network connectivity](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.11-Network-connectivity/README.md)
- [1.12 Correlation and remediation](docs/1.0-Perimeter-Security-and-Intrusion-Prevention/1.12-Correlation-and-remediation/README.md)

---

## 2.0 Secure Connectivity and Segmentation
- [2.1 AnyConnect RA VPN](docs/2.0-Secure-Connectivity-and-Segmentation/2.1-AnyConnect-RA-VPN/README.md)
- [2.2 IOS CA](docs/2.0-Secure-Connectivity-and-Segmentation/2.2-IOS-CA/README.md)
- [2.3 FlexVPN / DMVPN / IPsec](docs/2.0-Secure-Connectivity-and-Segmentation/2.3-FlexVPN-DMVPN-IPsec/README.md)
- [2.4 VPN HA](docs/2.0-Secure-Connectivity-and-Segmentation/2.4-VPN-HA/README.md)
- [2.5 Infrastructure segmentation](docs/2.0-Secure-Connectivity-and-Segmentation/2.5-Infrastructure-segmentation/README.md)
- [2.6 TrustSec / SGT / SXP](docs/2.0-Secure-Connectivity-and-Segmentation/2.6-TrustSec-SGT-SXP/README.md)

---

## 3.0 Security Infrastructure
- [3.1 Device hardening](docs/3.0-Security-Infrastructure/3.1-Device-hardening/README.md)
- [3.2 Management plane](docs/3.0-Security-Infrastructure/3.2-Management-plane/README.md)
- [3.3 Data plane](docs/3.0-Security-Infrastructure/3.3-Data-plane/README.md)
- [3.4 L2 security](docs/3.0-Security-Infrastructure/3.4-L2-security/README.md)
- [3.5 Wireless security](docs/3.0-Security-Infrastructure/3.5-Wireless-security/README.md)
- [3.6 Monitoring](docs/3.0-Security-Infrastructure/3.6-Monitoring/README.md)
- [3.7 Compliance](docs/3.0-Security-Infrastructure/3.7-Compliance/README.md)
- [3.8 Cisco SAFE](docs/3.0-Security-Infrastructure/3.8-Cisco-SAFE/README.md)
- [3.9 APIs](docs/3.0-Security-Infrastructure/3.9-APIs/README.md)
- [3.10 DNAC APIs](docs/3.0-Security-Infrastructure/3.10-DNAC-APIs/README.md)

---

## 4.0 Identity Management, Information Exchange, and Access Control
- [4.1 ISE scalability](docs/4.0-Identity-Management/4.1-ISE-scalability/README.md)
- [4.2 Network access AAA](docs/4.0-Identity-Management/4.2-Network-access-AAA/README.md)
- [4.3 Admin access](docs/4.0-Identity-Management/4.3-Admin-access/README.md)
- [4.4 802.1X / MAB](docs/4.0-Identity-Management/4.4-802.1X-MAB/README.md)
- [4.5 Guest lifecycle](docs/4.0-Identity-Management/4.5-Guest-lifecycle/README.md)
- [4.6 BYOD](docs/4.0-Identity-Management/4.6-BYOD/README.md)
- [4.7 External identity](docs/4.0-Identity-Management/4.7-External-identity/README.md)
- [4.8 AnyConnect provisioning](docs/4.0-Identity-Management/4.8-AnyConnect-provisioning/README.md)
- [4.9 Posture](docs/4.0-Identity-Management/4.9-Posture/README.md)
- [4.10 Profiling](docs/4.0-Identity-Management/4.10-Profiling/README.md)
- [4.11 MDM](docs/4.0-Identity-Management/4.11-MDM/README.md)
- [4.12 Certificate authentication](docs/4.0-Identity-Management/4.12-Cert-auth/README.md)
- [4.13 Authentication methods](docs/4.0-Identity-Management/4.13-Auth-methods/README.md)
- [4.14 Identity mapping](docs/4.0-Identity-Management/4.14-Identity-mapping/README.md)
- [4.15 pxGrid](docs/4.0-Identity-Management/4.15-pxGrid/README.md)
- [4.16 MFA](docs/4.0-Identity-Management/4.16-MFA/README.md)
- [4.17 DUO](docs/4.0-Identity-Management/4.17-DUO/README.md)
- [4.18 IBNS 2.0](docs/4.0-Identity-Management/4.18-IBNS2.0/README.md)

---

## 5.0 Advanced Threat Protection and Content Security
- [5.1 AMP](docs/5.0-Advanced-Threat-Protection/5.1-AMP/README.md)
- [5.2 Malware analysis](docs/5.0-Advanced-Threat-Protection/5.2-Malware-analysis/README.md)
- [5.3 Packet capture](docs/5.0-Advanced-Threat-Protection/5.3-Packet-capture/README.md)
- [5.4 Cloud security](docs/5.0-Advanced-Threat-Protection/5.4-Cloud-security/README.md)
- [5.5 Web filtering](docs/5.0-Advanced-Threat-Protection/5.5-Web-filtering/README.md)
- [5.6 WCCP](docs/5.0-Advanced-Threat-Protection/5.6-WCCP/README.md)
- [5.7 Email security](docs/5.0-Advanced-Threat-Protection/5.7-Email-security/README.md)
- [5.8 HTTP decryption](docs/5.0-Advanced-Threat-Protection/5.8-HTTP-decryption/README.md)
- [5.9 SMA](docs/5.0-Advanced-Threat-Protection/5.9-SMA/README.md)
- [5.10 Threat solutions](docs/5.0-Advanced-Threat-Protection/5.10-Threat-solutions/README.md)

---

# 🧭 Navigation
- [docs/](docs/) — 全 Blueprint の学習メモ  
- GitHub Pages で閲覧する場合は `/docs/` が自動的にサイト化されます

---

