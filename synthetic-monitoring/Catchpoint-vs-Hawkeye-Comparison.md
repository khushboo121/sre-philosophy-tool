# Catchpoint vs Hawkeye: Synthetic Monitoring Comparison

**Document Created:** January 2025  


## Executive Summary

*[To be filled from Confluence pages]*

---

## 1. Overview

### 1.1 Catchpoint
*[Extract key information from the Catchpoint Confluence page]*

**Vendor:** Catchpoint Systems, Inc.  
**Type:** Internet Performance Monitoring (IPM) Platform  
**Focus:** Internet Stack visibility, synthetic monitoring, real user monitoring

**Key Differentiators:**
- Internet Synthetic Monitoring (beyond traditional synthetic monitoring)
- 3,000+ global intelligent agents
- Comprehensive Internet Stack monitoring (DNS, CDN, BGP, APIs, etc.)
- Long-term data retention (up to 7 years)

### 1.2 Hawkeye
*[Extract key information from the Hawkeye Confluence page]*

**Vendor:** Keysight Technologies (formerly Ixia)  
**Type:** Active Network Monitoring Platform  
**Focus:** Network performance validation, application testing, edge visibility

**Key Differentiators:**
- Hardware and software endpoint deployment options
- Unified communications testing (Skype for Business, Teams)
- Wi-Fi connectivity testing
- Inline monitoring capabilities with fail-to-wire

---

## 2. Architecture & Deployment

### 2.1 Catchpoint

**Deployment Model:**
- Cloud-based SaaS platform
- Global agent network (managed by Catchpoint)
- Enterprise agents available for on-premise deployment

**Agent Types:**
- Public cloud nodes
- Internet backbone nodes
- Last-mile nodes
- Wireless nodes
- On-premise nodes (enterprise agents)

**Network Coverage:**
- 3,000+ agents worldwide
- 350+ cities
- 108+ countries
- Diverse connection types (cloud, backbone, last-mile, wireless)

### 2.2 Hawkeye

**Deployment Model:**
- On-premise web server (central location: NOC, data center)
- Can be deployed as VM (VMware, KVM, AWS)
- Physical appliance option (Vision Management Appliance)

**Endpoint Types:**
- **Hardware Endpoints:**
  - Vision E15 (packet aggregation, NetFlow, packet capture)
  - IxProbe (inline monitoring, fail-to-wire, 1G throughput)
  - XRPi (Raspberry Pi-based, Wi-Fi enabled, portable)
- **Software Endpoints:**
  - VM endpoints
  - Docker containers
  - Software agents

**Deployment Locations:**
- Customer premises
- Mobile devices (Wi-Fi or Cellular)
- Remote sites and head offices
- Network aggregation points (PEs)
- Core network, MPLS routers
- Data centers
- Virtual machines and servers
- Public cloud (AWS, Azure)

---

## 3. Monitoring Capabilities

### 3.1 Catchpoint

**Synthetic Monitoring Types:**
- User Journey Transaction Testing (multi-step workflows)
- HTTP/HTTPS Monitoring
- Real Browser Testing (Chrome, Firefox, Safari, etc.)
- Mobile Simulation (Android, iOS, various network speeds)
- API Monitoring (REST, SOAP, GraphQL)
- DNS Monitoring (comprehensive DNS resolution tracking)
- CDN Monitoring (cache status, PoP mapping, routing)
- BGP Monitoring (route analytics, hijack detection)
- SSL/TLS Certificate Monitoring
- WebSocket Testing
- SMTP/POP3/IMAP Testing
- FTP Monitoring
- NTP Server Testing
- MQTT Testing (IoT devices)

**Internet Stack Monitoring:**
- DNS resolution performance
- CDN performance and cache behavior
- BGP route monitoring
- ISP performance
- Cloud service provider monitoring
- Third-party API dependencies

**Real User Monitoring (RUM):**
- Browser-based RUM
- Network Error Logging (NEL)
- Session replay capabilities

### 3.2 Hawkeye

**Test Types:**
- **Network Performance:**
  - Speed tests (site-to-site)
  - Bandwidth availability/verification (bit blasting, TCP-based)
  - IP network SLA verification (one-way delay, jitter, packet loss)
  - Echo tests (ICMP, TCP, UDP)
  - Path discovery (hop-by-hop)
- **Application Testing:**
  - Unified communications (Skype for Business, Teams - Microsoft Certified)
  - Real-time streaming verification
  - Mean Opinion Score (MOS) for voice (G711, G729, AMR)
  - Media Delivery Index (MDI) for video streaming
  - User experience tests (web page downloads)
  - DNS response time
  - Netflix, YouTube, Dash/Adaptive streaming tests
  - Multicast video
- **Wi-Fi Testing:**
  - Wi-Fi network advanced metrics
  - Wi-Fi scanning and connectivity tests
  - Dual-band support (2.4GHz, 5GHz)
- **Class of Service (CoS):**
  - CoS implementation validation
  - Oversubscription scenarios
- **Other:**
  - Remote destination port opening verification
  - Synthetic traffic generation (up to 10G links)

---

## 4. Analytics & Reporting

### 4.1 Catchpoint

**Analytics Features:**
- Waterfall charts (granular HTTP metrics)
- Trend analysis (day-over-day, year-over-year)
- Data aggregation (min, max, average, median, 95th percentile)
- Scatterplot visualization
- Cumulative Distribution Function (CDF)
- Histogram/frequency distribution
- Advanced visualizations (Bubble Chart, Grid Chart, Heat Maps)
- AI-powered smartboards
- Internet Stack Map (dependency visualization)
- Internet Sonar (global outage awareness)

**Metrics:**
- 100+ standard metrics
- 30+ dimensions
- Custom metrics/dimensions support
- Core Web Vitals (LCP, CLS, TBT, FCP, TTI)
- Page load time, DNS lookup time, TTFB
- Speed Index

**Reporting:**
- Customizable dashboards
- Shareable dashboards (public links)
- Automated reports
- SLA management and validation
- Long-term data retention (up to 7 years)

### 4.2 Hawkeye

**Analytics Features:**
- Real-time analysis
- Geographic dashboards
- Outlier dashboard (machine learning-based)
- Time series analytics
- Pass/fail metrics
- Trend reports on historical data
- Network behavior analysis
- Path discovery visualization with geolocation

**Metrics:**
- Network quality metrics (latency, jitter, packet loss)
- Application performance metrics
- Quality of Experience (QoE) metrics
- Quality of Service (QoS) metrics
- Mean Opinion Score (MOS)
- Media Delivery Index (MDI)
- Throughput measurements
- Wi-Fi metrics

**Reporting:**
- Web-based platform for multi-user access
- Real-time result viewing
- Single instance or grouped test results
- Export capabilities
- Email reports
- Scheduled automatic reports
- Customizable notifications

---

## 5. Alerting & Notifications

### 5.1 Catchpoint

**Alerting Capabilities:**
- Real-time alerts
- Configurable thresholds
- Alert on raw or aggregated data
- Percentage-based alerting (concurrent or sequential failures)
- Trend change detection
- Transient incident handling
- False positive reduction
- Maintenance window support

**Notification Channels:**
- Email
- SMS
- Slack
- PagerDuty
- Opsgenie
- Webhooks
- API integrations
- Custom integrations

**Alert Customization:**
- Granular threshold configuration
- Transaction step-level alerting
- Multi-condition alerting
- Alert grouping and tagging

### 5.2 Hawkeye

**Alerting Capabilities:**
- Proactive fault detection
- Continuous interval testing
- Machine learning-based outlier detection
- Pass/fail metrics
- Threshold-based alerting
- Auto-learn baseline thresholds (ML algorithms)

**Notification Channels:**
- Email
- SNMP traps
- Customized notifications
- Webhooks (likely, to be confirmed)

---

## 6. Integration & APIs

### 6.1 Catchpoint

**Integrations:**
- ServiceNow
- Grafana
- Splunk
- Datadog
- PagerDuty
- Opsgenie
- Slack
- WebPageTest integration
- OpenTelemetry support

**API Capabilities:**
- REST API
- Webhook API
- Real-time data streaming
- Data export to data lakes
- Custom script support (Playwright, Puppeteer, Selenium)

**Data Export:**
- CSV/Excel export
- API-based data extraction
- Webhook-based real-time data delivery
- Integration with analytics platforms

### 4.2 Hawkeye

**Integrations:**
- *[To be filled from Confluence page]*

**API Capabilities:**
- *[To be filled from Confluence page]*

**Data Export:**
- Email reports
- Export functionality (format to be confirmed)
- NetFlow/IPFIX export (via Vision Edge 1S)

---

## 7. Security & Compliance

### 7.1 Catchpoint

**Security Features:**
- Encryption for critical business data
- Secure credential storage
- Single Sign-On (SSO) support
- User teams with varied permission levels
- Access privileges management
- Sub-tenant support

**Compliance:**
- *[To be filled from Confluence page]*

### 7.2 Hawkeye

**Security Features:**
- *[To be filled from Confluence page]*

**Compliance:**
- *[To be filled from Confluence page]*

---

## 8. Use Cases

### 8.1 Catchpoint

**Primary Use Cases:**
- Tier-1 Application Resilience
- Global Network Reachability
- Edge & Cloud Optimization
- AI Stack Resilience
- API Performance Monitoring
- Internal Network Monitoring
- Web Performance Monitoring
- Workforce Experience Monitoring
- CDN Performance Optimization
- Multi-CDN Comparison
- SLA Validation
- Internet outage detection and correlation

### 8.2 Hawkeye

**Primary Use Cases:**
- Network performance validation
- Unified communications testing
- Wi-Fi network monitoring
- Last-mile connectivity monitoring
- Branch office monitoring
- Service pre-launch assessment
- SLA verification
- Class of Service validation
- Real-time streaming verification
- Edge visibility and performance monitoring

---

## 9. Pricing & Licensing

### 9.1 Catchpoint

**Pricing Model:**
- *[To be filled from Confluence page]*
- Transparent, usage-aligned pricing
- No "use it or lose it" credit system

**Licensing:**
- *[To be filled from Confluence page]*

### 9.2 Hawkeye

**Pricing Model:**
- *[To be filled from Confluence page]*

**Licensing Options:**
- Solution bundles based on endpoint count:
  - 10 Endpoint Bundle
  - 25 Endpoint Bundle
  - 50 Endpoint Bundle
  - 100 Endpoint Bundle
  - 200 Endpoint Bundle
- Includes: central management system, endpoint registration, node-to-node test licenses, real services tests, user seat licenses

---

## 10. Support & Training

### 10.1 Catchpoint

**Support:**
- 24/7 tierless support
- Dedicated customer success team
- Expert services
- Five-time Stevie Award winner for customer service

**Training:**
- Online self-service training materials
- Custom training for groups
- Certification programs

### 10.2 Hawkeye

**Support:**
- *[To be filled from Confluence page]*

**Training:**
- *[To be filled from Confluence page]*

---

## 11. Strengths & Weaknesses

### 11.1 Catchpoint

**Strengths:**
- Largest and most diverse global monitoring network
- Comprehensive Internet Stack visibility
- Long-term data retention (7 years)
- Internet Synthetic Monitoring (unique capability)
- Strong BGP and DNS monitoring
- Real User Monitoring integration
- Extensive API and integration ecosystem
- AI-powered analytics

**Potential Weaknesses:**
- *[To be filled from Confluence page]*
- Primarily cloud-based (may not suit all on-premise requirements)
- Focus on Internet Stack may be overkill for simple internal network monitoring

### 11.2 Hawkeye

**Strengths:**
- Flexible deployment (hardware and software endpoints)
- Inline monitoring with fail-to-wire
- Unified communications testing (Microsoft Certified)
- Wi-Fi monitoring capabilities
- On-premise deployment option
- Cost-effective endpoint distribution
- Machine learning-based analytics

**Potential Weaknesses:**
- *[To be filled from Confluence page]*
- Requires infrastructure deployment and management
- May have limited global coverage compared to cloud-based solutions
- Focus on network/application testing rather than comprehensive Internet Stack monitoring

---

## 12. Comparison Matrix

| Feature | Catchpoint | Hawkeye |
|---------|-----------|---------|
| **Deployment Model** | Cloud-based SaaS | On-premise (VM or physical) |
| **Global Agent Network** | 3,000+ agents, 350+ cities, 108+ countries | Customer-deployed endpoints |
| **Internet Stack Monitoring** | ✅ Comprehensive (DNS, CDN, BGP, etc.) | ⚠️ Limited (focused on network/app) |
| **BGP Monitoring** | ✅ Advanced (1,600+ BGP peers) | ❌ Not mentioned |
| **CDN Monitoring** | ✅ Deep visibility | ⚠️ Basic |
| **Real User Monitoring** | ✅ Yes | ❌ No |
| **Unified Communications Testing** | ⚠️ Not specifically mentioned | ✅ Yes (Microsoft Certified) |
| **Wi-Fi Monitoring** | ⚠️ Limited | ✅ Advanced |
| **Inline Monitoring** | ❌ No | ✅ Yes (IxProbe) |
| **Data Retention** | Up to 7 years | *[To be confirmed]* |
| **API & Integrations** | ✅ Extensive | ⚠️ Limited |
| **Machine Learning Analytics** | ✅ AI-powered | ✅ Yes |
| **On-Premise Option** | ✅ Enterprise agents | ✅ Full on-premise |
| **Hardware Endpoints** | ⚠️ Limited | ✅ Multiple options |
| **SLA Management** | ✅ Yes | ✅ Yes |
| **Mobile Testing** | ✅ Yes | ✅ Yes (XRPi) |

---

## 13. Recommendations

### When to Choose Catchpoint:
- Need comprehensive Internet Stack visibility (DNS, CDN, BGP, etc.)
- Require global monitoring without infrastructure management
- Need long-term data retention for compliance/trending
- Want integrated Real User Monitoring
- Focus on external-facing applications and services
- Need extensive third-party integrations
- Require BGP route monitoring and hijack detection

### When to Choose Hawkeye:
- Need on-premise deployment for security/compliance
- Require inline monitoring with fail-to-wire
- Focus on internal network and application performance
- Need unified communications testing (Teams, Skype)
- Require advanced Wi-Fi monitoring
- Want flexible hardware/software endpoint options
- Need to monitor last-mile connectivity and branch offices
- Prefer infrastructure ownership and control

---

## 14. Additional Notes

*[Add any additional information from the Confluence pages]*

---

## 15. References

- Catchpoint Confluence Page: SYM/pages/318374474/Catchpoint
- Hawkeye Confluence Page: SYM/pages/310881591/Hawkeye
- Catchpoint Official Website: https://www.catchpoint.com
- Hawkeye Product Information: https://www.keysight.com (Ixia/Keysight)

---

**Document Status:** Template created - requires information from Confluence pages to complete



