# SOC System using Wazuh (SIEM/EDR) Integrated with ELK Stack

**[English](#english) | [Tiếng Việt](#tiếng-việt)**

---

# English

**Internship report – Information Security**
**Topic:** Research on a SOC system that uses SIEM/EDR (Wazuh) as the platform, integrated with ELK Stack

- **Student:** Ngo Duc Thang (ID: B2203737)
- **Supervisor:** Dr. Ngo Ba Hung
- **Internship place:** VNPT Can Tho – Solution Business Center (12/01/2026 – 18/04/2026)
- **Company mentor:** Huynh Quoc Vinh

## Contents

- [1. Introduction](#1-introduction)
- [2. Architecture](#2-architecture)
- [3. Lab network](#3-lab-network)
- [4. Requirements](#4-requirements)
- [5. Installation](#5-installation)
- [6. Agents and pfSense](#6-agents-and-pfsense)
- [7. Custom detection rules](#7-custom-detection-rules)
- [8. Testing](#8-testing)
- [9. Results and future work](#9-results-and-future-work)
- [10. Notes](#10-notes)

## 1. Introduction

A SOC needs to watch logs from many places. Doing this by hand is not possible. So this project builds a small SOC lab with **Wazuh** as the SIEM/EDR platform and **ELK Stack** (Elasticsearch, Logstash, Kibana) for storing and showing alerts.

What I did in this repo:

- Install and configure Wazuh Manager and ELK Stack on one server.
- Deploy Wazuh Agent on Ubuntu and Windows 10.
- Add pfSense as router, firewall and an agentless log source (syslog).
- Write custom rules and correlation to detect SSH brute-force and TCP flood.
- Test every rule with real attacks and check the alerts on Kibana.

This is a learning lab. It is not a production system.

## 2. Architecture

The pipeline has 5 layers:

| Layer | Component | Job |
|---|---|---|
| Collection | Wazuh Agent / syslog | Collect logs from endpoints and network devices |
| Analysis | Wazuh Manager | Decode logs, match rules, create alerts with MITRE ATT&CK tags |
| Forwarding | Logstash | Send alerts to Elasticsearch |
| Storage | Elasticsearch | Index and search alerts |
| Visualization | Kibana | Dashboards and alerting |

Data flow:

```
Wazuh Manager
   → /var/ossec/logs/alerts/alerts.json
        ↓
     Logstash
        ↓
   Elasticsearch
        ↓
      Kibana
```

Why use both? Wazuh is strong at security analysis (decoders, rules, MITRE mapping). ELK is strong at storing, searching and visualizing data. ELK alone has no threat detection rules, so it needs Wazuh.

## 3. Lab network

![SOC lab network diagram](images/network-diagram.png)

pfSense is the router between three LAN branches. It is also the firewall. It sends its logs to the Manager because it cannot run a Wazuh Agent.

| Device | Role | IP address | Gateway |
|---|---|---|---|
| pfSense (WAN) | Router / firewall / syslog source | 10.0.2.15/24 | - |
| pfSense (LAN1) | Gateway of LAN1 | 192.168.10.1/24 | - |
| pfSense (LAN2) | Gateway of LAN2 | 192.168.20.1/24 | - |
| pfSense (LAN3) | Gateway of LAN3 | 192.168.30.1/24 | - |
| Wazuh Manager + ELK | SIEM server | 192.168.10.105/24 | 192.168.10.1 |
| Ubuntu | Wazuh Agent | 192.168.20.103/24 | 192.168.20.1 |
| Windows 10 | Wazuh Agent | 192.168.30.104/24 | 192.168.30.1 |

LAN1 is the "LAN manager". You can open the pfSense web interface from the Manager machine through its gateway IP `192.168.10.1`.

## 4. Requirements

| Software | Version |
|---|---|
| Elasticsearch | 7.10.2 |
| Kibana | 7.10.2 |
| Logstash | 7.10.2 (`1:7.10.2-1`) |
| Wazuh Manager | 4.3.11-1 |
| Wazuh Agent | 4.3.11 |
| Server OS | Ubuntu (apt-based) |

Attack tools for testing: `hydra` (SSH brute-force) and `hping3` (flood). A Kali Linux machine can be used as the attacker.

## 5. Installation

Everything below runs on the Manager machine (`192.168.10.105`).

### 5.1. Elasticsearch

```bash
# dependencies
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg

# Elastic 7.x repository
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
| sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
https://artifacts.elastic.co/packages/7.x/apt stable main" \
| sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install -y elasticsearch=7.10.2
```

Edit `/etc/elasticsearch/elasticsearch.yml` (single node):

```yaml
cluster.name: wazuh-cluster
node.name: wazuh-node
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
```

Start and test:

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

curl http://localhost:9200
```

### 5.2. Kibana

```bash
sudo apt install -y kibana=7.10.2
```

Edit `/etc/kibana/kibana.yml`:

```yaml
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

### 5.3. Logstash

```bash
sudo apt install -y logstash=1:7.10.2-1
```

Create the pipeline `/etc/logstash/conf.d/wazuh-alerts.conf`:

```conf
input {
  file {
    path => "/var/ossec/logs/alerts/alerts.json"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => json
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "wazuh-alerts-%{+YYYY.MM.dd}"
  }
}
```

Give Logstash permission to read `alerts.json`:

```bash
sudo usermod -a -G wazuh logstash
sudo systemctl restart logstash
```

Check the pipeline syntax and start Logstash:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

sudo systemctl enable logstash
sudo systemctl start logstash
```

### 5.4. Wazuh Manager

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
| sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
https://packages.wazuh.com/4.x/apt stable main" \
| sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
sudo apt install -y wazuh-manager=4.3.11-1

sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager

sudo /var/ossec/bin/wazuh-control status
```

### 5.5. Check the data flow

```bash
# 1. Does Wazuh create alerts?
ls -l /var/ossec/logs/alerts/alerts.json

# 2. Does the index exist in Elasticsearch?
curl http://localhost:9200/_cat/indices?v
```

You must see an index like `wazuh-alerts-YYYY.MM.dd`.

Then open Kibana at `http://<IP_SERVER>:5601` and create an index pattern:

- Index pattern: `wazuh-alerts-*`
- Time field: `@timestamp`

## 6. Agents and pfSense

### 6.1. Wazuh Agent

Install Wazuh Agent 4.3.11 on the Ubuntu and Windows machines. Point them to the Manager `192.168.10.105`.

Wazuh Agent 4.3.11 does not collect OpenSSH logs on Windows by default. Add this block at the end of `ossec.conf` on the Windows agent, then restart the agent:

```xml
<localfile>
  <location>OpenSSH/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### 6.2. pfSense (agentless)

pfSense cannot run a Wazuh Agent, so its logs go to the Manager by syslog.

- Install the **syslog-ng** package on pfSense. It collects, sorts, stores and forwards logs. The output is cleaner than the default pfSense log.
- Logs are sent to the Manager (default syslog port 514, and syslog-ng forwards to port 5140 in this lab).
- pfSense also protects the LAN branches as a firewall.

## 7. Custom detection rules

All rules go into `/var/ossec/etc/rules/local_rules.xml` on the Manager. After editing, restart the Manager:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

You can test a rule before using it:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

### 7.1. Ubuntu: SSH monitoring (100100 – 100104)

| Rule ID | Level | Detects |
|---|---|---|
| 100100 | 5 | SSH login successful |
| 100101 | 7 | SSH login failed |
| 100102 | 8 | SSH login with invalid (non-existent) user |
| 100103 | 12 | SSH brute force (5 failed logins in 60s) |
| 100104 | 15 | SSH brute force succeeded (account compromised) |

The brute-force chain: 5 failed logins in 60s → a successful login from the same source IP → alert for possible compromised account.

```xml
<group name="custom_ssh_monitoring,ssh,authentication,">

  <!-- SSH Login Success -->
  <rule id="100100" level="5">
    <if_sid>5762</if_sid>
    <description>SSH login successful</description>
    <mitre>
      <id>T1078</id>
    </mitre>
    <group>authentication_success,ssh,</group>
  </rule>

  <!-- SSH Login Failed -->
  <rule id="100101" level="7">
    <if_sid>5760</if_sid>
    <description>SSH login failed</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,ssh,</group>
  </rule>

  <!-- SSH Invalid User Attempt -->
  <rule id="100102" level="8">
    <if_sid>5761</if_sid>
    <description>SSH login attempt with invalid user</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,ssh,</group>
  </rule>

  <!-- SSH Brute Force -->
  <rule id="100103" level="12" frequency="5" timeframe="60">
    <if_matched_sid>100101</if_matched_sid>
    <description>Possible SSH brute force attack detected</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication,ssh,attack,</group>
  </rule>

  <!-- Success after brute force -->
  <rule id="100104" level="15" timeframe="120">
    <if_matched_sid>100103</if_matched_sid> <!-- already brute forced -->
    <if_sid>5715</if_sid>                   <!-- login success -->
    <same_source_ip />
    <same_user />
    <description>SSH brute force succeeded (account compromised)</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication,ssh,attack,compromise,</group>
  </rule>

</group>
```

### 7.2. Windows 10: OpenSSH monitoring (100300 – 100305)

| Rule ID | Level | Detects |
|---|---|---|
| 100300 | 0 | Parent rule: SSH login failed from `sshd.exe` (Security log) |
| 100301 | 7 | SSH wrong password |
| 100302 | 7 | SSH wrong username (user does not exist) |
| 100303 | 12 | SSH brute force (5 failed logins in 60s, same user) |
| 100304 | 4 | SSH login successful |
| 100305 | 15 | Suspected brute force success (100303 + 100304, same user) |

```xml
<group name="windows,openssh,authentication,">

  <!-- Parent rule: SSH failed from sshd.exe via Security log -->
  <rule id="100300" level="0">
    <if_sid>60122</if_sid>
    <field name="win.eventdata.processName" type="pcre2">(?i)sshd\.exe</field>
    <description>Windows OpenSSH: SSH login failed (Security log)</description>
  </rule>

  <!-- Wrong password -->
  <rule id="100301" level="7">
    <if_sid>100300</if_sid>
    <field name="win.eventdata.subStatus">^0xc000006a$</field>
    <description>Windows OpenSSH: SSH wrong password - user $(win.eventdata.targetUserName)</description>
  </rule>

  <!-- User does not exist -->
  <rule id="100302" level="7">
    <if_sid>100300</if_sid>
    <field name="win.eventdata.subStatus">^0xc0000064$</field>
    <description>Windows OpenSSH: SSH wrong username - $(win.eventdata.targetUserName) does not exist</description>
  </rule>

  <!-- Brute force: 5 failures in 60s, same user -->
  <rule id="100303" level="12" frequency="5" timeframe="60">
    <if_matched_sid>100300</if_matched_sid>
    <same_field>win.eventdata.targetUserName</same_field>
    <description>Windows OpenSSH: SSH brute force - user $(win.eventdata.targetUserName)</description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
  </rule>

  <!-- SSH login success via sshd.exe -->
  <rule id="100304" level="4">
    <if_sid>60106</if_sid>
    <field name="win.eventdata.processName" type="pcre2">(?i)sshd\.exe</field>
    <field name="win.system.eventID">^4624$</field>
    <description>Windows OpenSSH: SSH login successful - user $(win.eventdata.targetUserName)</description>
  </rule>

  <!-- Suspicious: brute force then successful login, same user, within 120s -->
  <rule id="100305" level="15" timeframe="120">
    <if_matched_sid>100303</if_matched_sid>
    <if_matched_sid>100304</if_matched_sid>
    <same_field>win.eventdata.targetUserName</same_field>
    <description>Windows OpenSSH: Suspected successful brute force - user $(win.eventdata.targetUserName)</description>
    <mitre>
      <id>T1110.001</id>
      <id>T1078</id>
    </mitre>
  </rule>

</group>
```

### 7.3. pfSense: flood detection (100500 – 100503)

The goal is to detect flood attacks into the management network (LAN1).

| Rule ID | Level | Detects |
|---|---|---|
| 100500 | 0 | Base rule for pfSense events |
| 100501 | 7 | Firewall block event |
| 100502 | 12 | Many blocks from the same source (10 in 60s) |
| 100503 | 14 | TCP flood blocked (20 blocks in 30s, same source) |

```xml
<group name="local,pfsense,custom">

  <rule id="100500" level="0">
    <if_sid>87700</if_sid>
    <description>Custom pfSense base rule.</description>
    <group>pfsense_custom,</group>
  </rule>

  <rule id="100501" level="7">
    <if_sid>100500</if_sid>
    <action>block</action>
    <description>Custom pfSense firewall block event.</description>
    <group>firewall_block_custom,pfsense,</group>
  </rule>

  <rule id="100502" level="12" frequency="10" timeframe="60" ignore="300">
    <if_matched_sid>100501</if_matched_sid>
    <same_source_ip />
    <description>Custom multiple pfSense blocks from same source.</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>multiple_blocks_custom,pfsense,</group>
  </rule>

  <rule id="100503" level="14" frequency="20" timeframe="30" ignore="120">
    <if_matched_sid>100501</if_matched_sid>
    <same_source_ip />
    <description>pfSense TCP flood detected (multiple blocks from same source)</description>
    <mitre>
      <id>T1498</id>
    </mitre>
    <group>pfsense,firewall,dos,flood,</group>
  </rule>

</group>
```

## 8. Testing

Use the Manager machine (or a Kali machine) as the attacker. After each test, check the alerts on Kibana.

| Test | How | Expected rule |
|---|---|---|
| Valid SSH login | `ssh user@<target>` with the right password | 100100 (Ubuntu) / 100304 (Windows) |
| Wrong password | `ssh user@<target>` with a wrong password | 100101 / 100301 |
| Wrong username | `ssh fakeuser@<target>` | 100102 / 100302 |
| SSH brute force | `hydra` with a wordlist | 100103 / 100303 |
| Brute force success | Fail 5 times, then login right on the 6th try, or `hydra` with the correct password in the list | 100104 / 100305 |
| TCP flood | Attacker in LAN3 (`192.168.30.0/24`) uses `hping3` against the Manager | 100501 → 100502 → 100503 |

Example commands:

```bash
# SSH brute force
hydra -l <username> -P passwords.txt ssh://<target_ip>

# TCP flood
sudo hping3 -S --flood -p <port> <target_ip>
```

For the flood test, pfSense has a rule to block TCP from the attacker to the LAN manager. The blocked connections create the pfSense logs that trigger rules 100501, 100502 and 100503.

## 9. Results and future work

**Results**

- All custom rules work. They detect valid, invalid and brute-force access.
- Kibana shows all the alerts from the custom rules.
- The full pipeline runs: Agent / syslog → Manager → Logstash → Elasticsearch → Kibana.
- Dashboards and basic alerts (email / Telegram) are set up on Kibana.
- The system was checked against MITRE ATT&CK, ISO and NIST.

**Not done yet**

- Only basic SSH attacks are covered. HTTP/HTTPS, RDP, web apps, lateral movement and privilege escalation are not monitored well.
- Alerts only notify (email, Telegram). Wazuh Active Response is not used, so the system cannot block IPs or isolate endpoints automatically.
- There is still noisy log. Rules and thresholds need tuning for a real environment.
- ISO/IEC 27001 and NIST 800-53 are not fully satisfied. More rules are needed for the missing controls.

## 10. Notes

- Keep the versions in section 4. Wazuh Agent and Manager should be the same version. Different versions can cause errors that are very hard to find.
- Do not skip the `usermod` step for Logstash. Without it Logstash cannot read `alerts.json` and nothing reaches Elasticsearch. There is no big error, so it is easy to miss.
- Restart the Wazuh Agent after editing `ossec.conf`. If not, the new log source does not work and you will wonder why there is no alert.
- Make sure the Manager listens for syslog on the same port you set in pfSense, and the firewall allows it. Wrong port means logs never arrive.
- In the flood demo I block the attacker IP on pfSense directly because it is fast to test. In a real network, block by other subnets or by a specific port, not all TCP.
- Use `wazuh-logtest` first. Paste the raw log line and see which rule fires. It saves a lot of time compared with restarting the Manager again and again.
- Noisy logs were the most annoying part. Too many false alerts hide the real ones. Tuning thresholds is slow, but it is really important.
- The first time I saw an alert appear in Kibana, I was really happy. All the setup before it felt worth it.
- IP addresses in this repo are from my lab. Please change them to fit your network.
- Only attack your own lab. Do not use `hydra` or `hping3` on real systems without permission.
- This is a small lab. It is not a full SOC.

---

# Tiếng Việt

**Báo cáo thực tập – Ngành An toàn thông tin**
**Đề tài:** Nghiên cứu hệ thống SOC sử dụng SIEM/EDR (Wazuh) làm nền tảng, tích hợp ELK Stack

- **Sinh viên:** Ngô Đức Thắng (MSSV: B2203737)
- **Giảng viên phụ trách:** TS. Ngô Bá Hùng
- **Nơi thực tập:** Trung tâm Kinh doanh Giải pháp VNPT Cần Thơ (12/01/2026 – 18/04/2026)
- **Cán bộ hướng dẫn:** Huỳnh Quốc Vinh

## Mục lục

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Kiến trúc](#2-kiến-trúc)
- [3. Sơ đồ mạng lab](#3-sơ-đồ-mạng-lab)
- [4. Yêu cầu](#4-yêu-cầu)
- [5. Cài đặt](#5-cài-đặt)
- [6. Agent và pfSense](#6-agent-và-pfsense)
- [7. Rule phát hiện tự viết](#7-rule-phát-hiện-tự-viết)
- [8. Kiểm thử](#8-kiểm-thử)
- [9. Kết quả và hướng phát triển](#9-kết-quả-và-hướng-phát-triển)
- [10. Lưu ý](#10-lưu-ý)

## 1. Giới thiệu

SOC cần theo dõi log từ rất nhiều nơi. Làm thủ công thì không thể. Vì vậy đề tài xây dựng một lab SOC nhỏ, dùng **Wazuh** làm nền tảng SIEM/EDR và **ELK Stack** (Elasticsearch, Logstash, Kibana) để lưu trữ và hiển thị cảnh báo.

Trong repo này em đã làm:

- Cài đặt và cấu hình Wazuh Manager và ELK Stack trên một máy chủ.
- Triển khai Wazuh Agent trên Ubuntu và Windows 10.
- Thêm pfSense làm router, tường lửa và nguồn log không cần agent (syslog).
- Viết rule và correlation để phát hiện SSH brute-force và TCP flood.
- Test từng rule bằng tấn công thật và kiểm tra alert trên Kibana.

Đây là lab học tập, không phải hệ thống chạy thực tế.

## 2. Kiến trúc

Pipeline gồm 5 tầng:

| Tầng | Thành phần | Nhiệm vụ |
|---|---|---|
| Thu thập | Wazuh Agent / syslog | Thu log từ endpoint và thiết bị mạng |
| Phân tích | Wazuh Manager | Giải mã log, so khớp rule, sinh cảnh báo kèm MITRE ATT&CK |
| Chuyển tiếp | Logstash | Đẩy cảnh báo sang Elasticsearch |
| Lưu trữ | Elasticsearch | Lập chỉ mục và tìm kiếm cảnh báo |
| Trực quan hóa | Kibana | Dashboard và cảnh báo |

Luồng dữ liệu:

```
Wazuh Manager
   → /var/ossec/logs/alerts/alerts.json
        ↓
     Logstash
        ↓
   Elasticsearch
        ↓
      Kibana
```

Vì sao dùng cả hai? Wazuh mạnh về phân tích an ninh (decoder, rule, ánh xạ MITRE). ELK mạnh về lưu trữ, tìm kiếm và trực quan hóa dữ liệu. ELK đứng một mình thì không có rule phát hiện mối đe dọa, nên cần kết hợp với Wazuh.

## 3. Sơ đồ mạng lab

![Sơ đồ mạng lab SOC](images/network-diagram.png)

pfSense là router giữa ba nhánh LAN. Nó cũng là tường lửa. pfSense gửi log về Manager vì không cài được Wazuh Agent.

| Thiết bị | Vai trò | Địa chỉ IP | Gateway |
|---|---|---|---|
| pfSense (WAN) | Router / tường lửa / nguồn syslog | 10.0.2.15/24 | - |
| pfSense (LAN1) | Gateway của LAN1 | 192.168.10.1/24 | - |
| pfSense (LAN2) | Gateway của LAN2 | 192.168.20.1/24 | - |
| pfSense (LAN3) | Gateway của LAN3 | 192.168.30.1/24 | - |
| Wazuh Manager + ELK | Máy chủ SIEM | 192.168.10.105/24 | 192.168.10.1 |
| Ubuntu | Wazuh Agent | 192.168.20.103/24 | 192.168.20.1 |
| Windows 10 | Wazuh Agent | 192.168.30.104/24 | 192.168.30.1 |

LAN1 là "LAN manager". Có thể mở giao diện web của pfSense ngay từ máy Manager thông qua IP gateway `192.168.10.1`.

## 4. Yêu cầu

| Phần mềm | Phiên bản |
|---|---|
| Elasticsearch | 7.10.2 |
| Kibana | 7.10.2 |
| Logstash | 7.10.2 (`1:7.10.2-1`) |
| Wazuh Manager | 4.3.11-1 |
| Wazuh Agent | 4.3.11 |
| Hệ điều hành máy chủ | Ubuntu (dùng apt) |

Công cụ tấn công để test: `hydra` (SSH brute-force) và `hping3` (flood). Có thể dùng máy Kali Linux làm máy tấn công.

## 5. Cài đặt

Tất cả lệnh bên dưới chạy trên máy Manager (`192.168.10.105`).

### 5.1. Elasticsearch

```bash
# cài các gói phụ thuộc
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg

# thêm repository Elastic 7.x
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
| sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
https://artifacts.elastic.co/packages/7.x/apt stable main" \
| sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install -y elasticsearch=7.10.2
```

Sửa file `/etc/elasticsearch/elasticsearch.yml` (single node):

```yaml
cluster.name: wazuh-cluster
node.name: wazuh-node
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
```

Khởi động và kiểm tra:

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

curl http://localhost:9200
```

### 5.2. Kibana

```bash
sudo apt install -y kibana=7.10.2
```

Sửa file `/etc/kibana/kibana.yml`:

```yaml
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

### 5.3. Logstash

```bash
sudo apt install -y logstash=1:7.10.2-1
```

Tạo pipeline `/etc/logstash/conf.d/wazuh-alerts.conf`:

```conf
input {
  file {
    path => "/var/ossec/logs/alerts/alerts.json"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => json
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "wazuh-alerts-%{+YYYY.MM.dd}"
  }
}
```

Cấp quyền đọc file `alerts.json` cho Logstash:

```bash
sudo usermod -a -G wazuh logstash
sudo systemctl restart logstash
```

Kiểm tra cú pháp pipeline rồi khởi động Logstash:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

sudo systemctl enable logstash
sudo systemctl start logstash
```

### 5.4. Wazuh Manager

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
| sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
https://packages.wazuh.com/4.x/apt stable main" \
| sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
sudo apt install -y wazuh-manager=4.3.11-1

sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager

sudo /var/ossec/bin/wazuh-control status
```

### 5.5. Kiểm tra luồng dữ liệu

```bash
# 1. Wazuh có sinh alert không?
ls -l /var/ossec/logs/alerts/alerts.json

# 2. Index đã có trong Elasticsearch chưa?
curl http://localhost:9200/_cat/indices?v
```

Phải thấy index dạng `wazuh-alerts-YYYY.MM.dd`.

Sau đó mở Kibana tại `http://<IP_SERVER>:5601` và tạo index pattern:

- Index pattern: `wazuh-alerts-*`
- Time field: `@timestamp`

## 6. Agent và pfSense

### 6.1. Wazuh Agent

Cài Wazuh Agent 4.3.11 trên máy Ubuntu và Windows. Trỏ agent về Manager `192.168.10.105`.

Wazuh Agent 4.3.11 không thu log OpenSSH trên Windows theo mặc định. Thêm đoạn sau vào cuối file `ossec.conf` của agent Windows, rồi restart agent:

```xml
<localfile>
  <location>OpenSSH/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### 6.2. pfSense (không dùng agent)

pfSense không cài được Wazuh Agent, nên log được gửi về Manager qua syslog.

- Cài gói **syslog-ng** trên pfSense. Gói này thu thập, phân loại, lưu trữ và chuyển tiếp log. Log đầu ra dễ đọc hơn log mặc định của pfSense.
- Log được gửi về Manager (cổng syslog mặc định là 514, và trong lab này syslog-ng chuyển tiếp qua cổng 5140).
- pfSense cũng bảo vệ các nhánh LAN với vai trò tường lửa.

## 7. Rule phát hiện tự viết

Tất cả rule được thêm vào `/var/ossec/etc/rules/local_rules.xml` trên Manager. Sau khi sửa, restart Manager:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

Có thể test rule trước khi dùng:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

### 7.1. Ubuntu: giám sát SSH (100100 – 100104)

| Rule ID | Level | Phát hiện |
|---|---|---|
| 100100 | 5 | Đăng nhập SSH thành công |
| 100101 | 7 | Đăng nhập SSH thất bại |
| 100102 | 8 | Đăng nhập SSH bằng user không tồn tại |
| 100103 | 12 | SSH brute force (5 lần thất bại trong 60s) |
| 100104 | 15 | SSH brute force thành công (tài khoản bị chiếm) |

Chuỗi phát hiện brute force: 5 lần đăng nhập thất bại trong 60s → có đăng nhập thành công từ cùng IP nguồn → gửi alert nghi ngờ tài khoản bị chiếm.

```xml
<group name="custom_ssh_monitoring,ssh,authentication,">

  <!-- SSH Login Success -->
  <rule id="100100" level="5">
    <if_sid>5762</if_sid>
    <description>SSH login successful</description>
    <mitre>
      <id>T1078</id>
    </mitre>
    <group>authentication_success,ssh,</group>
  </rule>

  <!-- SSH Login Failed -->
  <rule id="100101" level="7">
    <if_sid>5760</if_sid>
    <description>SSH login failed</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,ssh,</group>
  </rule>

  <!-- SSH Invalid User Attempt -->
  <rule id="100102" level="8">
    <if_sid>5761</if_sid>
    <description>SSH login attempt with invalid user</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,ssh,</group>
  </rule>

  <!-- SSH Brute Force -->
  <rule id="100103" level="12" frequency="5" timeframe="60">
    <if_matched_sid>100101</if_matched_sid>
    <description>Possible SSH brute force attack detected</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication,ssh,attack,</group>
  </rule>

  <!-- Success after brute force -->
  <rule id="100104" level="15" timeframe="120">
    <if_matched_sid>100103</if_matched_sid> <!-- đã brute force -->
    <if_sid>5715</if_sid>                   <!-- login thành công -->
    <same_source_ip />
    <same_user />
    <description>SSH brute force succeeded (account compromised)</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication,ssh,attack,compromise,</group>
  </rule>

</group>
```

### 7.2. Windows 10: giám sát OpenSSH (100300 – 100305)

| Rule ID | Level | Phát hiện |
|---|---|---|
| 100300 | 0 | Rule cha: SSH thất bại từ `sshd.exe` (Security log) |
| 100301 | 7 | SSH sai password |
| 100302 | 7 | SSH sai username (user không tồn tại) |
| 100303 | 12 | SSH brute force (5 lần thất bại trong 60s, cùng user) |
| 100304 | 4 | Đăng nhập SSH thành công |
| 100305 | 15 | Nghi ngờ brute force thành công (100303 + 100304, cùng user) |

```xml
<group name="windows,openssh,authentication,">

  <!-- Rule cha: nhận diện SSH failed từ sshd.exe qua Security log -->
  <rule id="100300" level="0">
    <if_sid>60122</if_sid>
    <field name="win.eventdata.processName" type="pcre2">(?i)sshd\.exe</field>
    <description>Windows OpenSSH: SSH login thất bại (Security log)</description>
  </rule>

  <!-- Sai password -->
  <rule id="100301" level="7">
    <if_sid>100300</if_sid>
    <field name="win.eventdata.subStatus">^0xc000006a$</field>
    <description>Windows OpenSSH: SSH sai password - user $(win.eventdata.targetUserName)</description>
  </rule>

  <!-- User không tồn tại -->
  <rule id="100302" level="7">
    <if_sid>100300</if_sid>
    <field name="win.eventdata.subStatus">^0xc0000064$</field>
    <description>Windows OpenSSH: SSH sai username - $(win.eventdata.targetUserName) không tồn tại</description>
  </rule>

  <!-- Brute-force: 5 lần thất bại trong 60 giây cùng user -->
  <rule id="100303" level="12" frequency="5" timeframe="60">
    <if_matched_sid>100300</if_matched_sid>
    <same_field>win.eventdata.targetUserName</same_field>
    <description>Windows OpenSSH: Brute-force SSH - user $(win.eventdata.targetUserName)</description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
  </rule>

  <!-- SSH login thành công qua sshd.exe -->
  <rule id="100304" level="4">
    <if_sid>60106</if_sid>
    <field name="win.eventdata.processName" type="pcre2">(?i)sshd\.exe</field>
    <field name="win.system.eventID">^4624$</field>
    <description>Windows OpenSSH: SSH login thành công - user $(win.eventdata.targetUserName)</description>
  </rule>

  <!-- Nghi ngờ: brute-force rồi login thành công cùng user trong 120 giây -->
  <rule id="100305" level="15" timeframe="120">
    <if_matched_sid>100303</if_matched_sid>
    <if_matched_sid>100304</if_matched_sid>
    <same_field>win.eventdata.targetUserName</same_field>
    <description>Windows OpenSSH: Nghi ngờ brute-force thành công - user $(win.eventdata.targetUserName)</description>
    <mitre>
      <id>T1110.001</id>
      <id>T1078</id>
    </mitre>
  </rule>

</group>
```

### 7.3. pfSense: phát hiện flood (100500 – 100503)

Mục tiêu là phát hiện các cuộc tấn công flood vào nhánh mạng quản lý (LAN1).

| Rule ID | Level | Phát hiện |
|---|---|---|
| 100500 | 0 | Rule gốc cho event của pfSense |
| 100501 | 7 | Event tường lửa chặn kết nối |
| 100502 | 12 | Nhiều lần bị chặn từ cùng một nguồn (10 lần trong 60s) |
| 100503 | 14 | Đã chặn TCP flood (20 lần chặn trong 30s, cùng nguồn) |

```xml
<group name="local,pfsense,custom">

  <rule id="100500" level="0">
    <if_sid>87700</if_sid>
    <description>Custom pfSense base rule.</description>
    <group>pfsense_custom,</group>
  </rule>

  <rule id="100501" level="7">
    <if_sid>100500</if_sid>
    <action>block</action>
    <description>Custom pfSense firewall block event.</description>
    <group>firewall_block_custom,pfsense,</group>
  </rule>

  <rule id="100502" level="12" frequency="10" timeframe="60" ignore="300">
    <if_matched_sid>100501</if_matched_sid>
    <same_source_ip />
    <description>Custom multiple pfSense blocks from same source.</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>multiple_blocks_custom,pfsense,</group>
  </rule>

  <rule id="100503" level="14" frequency="20" timeframe="30" ignore="120">
    <if_matched_sid>100501</if_matched_sid>
    <same_source_ip />
    <description>pfSense TCP flood detected (multiple blocks from same source)</description>
    <mitre>
      <id>T1498</id>
    </mitre>
    <group>pfsense,firewall,dos,flood,</group>
  </rule>

</group>
```

## 8. Kiểm thử

Dùng máy Manager (hoặc máy Kali) làm máy tấn công. Sau mỗi lần test, xem alert trên Kibana.

| Test | Cách làm | Rule mong đợi |
|---|---|---|
| SSH đúng | `ssh user@<target>` với đúng password | 100100 (Ubuntu) / 100304 (Windows) |
| Sai password | `ssh user@<target>` với password sai | 100101 / 100301 |
| Sai username | `ssh fakeuser@<target>` | 100102 / 100302 |
| SSH brute force | `hydra` với wordlist | 100103 / 100303 |
| Brute force thành công | Sai 5 lần rồi đăng nhập đúng ở lần thứ 6, hoặc `hydra` với wordlist có password đúng | 100104 / 100305 |
| TCP flood | Máy tấn công ở LAN3 (`192.168.30.0/24`) dùng `hping3` vào máy Manager | 100501 → 100502 → 100503 |

Lệnh ví dụ:

```bash
# SSH brute force
hydra -l <username> -P passwords.txt ssh://<target_ip>

# TCP flood
sudo hping3 -S --flood -p <port> <target_ip>
```

Với test flood, pfSense có một rule chặn TCP từ máy tấn công đến LAN manager. Các kết nối bị chặn tạo ra log pfSense, từ đó kích hoạt rule 100501, 100502 và 100503.

## 9. Kết quả và hướng phát triển

**Kết quả**

- Tất cả rule tự viết đều hoạt động. Phát hiện được truy cập hợp lệ, không hợp lệ và brute force.
- Kibana thấy được toàn bộ alert sinh ra từ các rule này.
- Pipeline chạy đủ: Agent / syslog → Manager → Logstash → Elasticsearch → Kibana.
- Đã có dashboard và cảnh báo cơ bản (email / Telegram) trên Kibana.
- Hệ thống đã được đánh giá theo MITRE ATT&CK, ISO và NIST.

**Chưa làm được**

- Mới chỉ bao phủ tấn công SSH cơ bản. Chưa giám sát tốt HTTP/HTTPS, RDP, ứng dụng web, tấn công chiều ngang và thăng quyền.
- Cảnh báo mới dừng ở thông báo (email, Telegram). Chưa dùng Active Response của Wazuh, nên hệ thống chưa tự chặn IP hay cô lập endpoint được.
- Vẫn còn log nhiễu. Cần tinh chỉnh rule và ngưỡng cảnh báo cho hợp với môi trường thực tế.
- ISO/IEC 27001 và NIST 800-53 chưa đáp ứng đầy đủ. Cần thêm rule cho các control còn thiếu.

## 10. Lưu ý

- Giữ đúng các phiên bản ở mục 4. Wazuh Agent và Manager nên cùng phiên bản. Khác phiên bản có thể gây lỗi rất khó tìm.
- Đừng bỏ qua bước `usermod` cho Logstash. Thiếu bước này Logstash không đọc được `alerts.json` và Elasticsearch sẽ không có dữ liệu. Hệ thống không báo lỗi lớn nên rất dễ bỏ sót.
- Nhớ restart Wazuh Agent sau khi sửa `ossec.conf`. Nếu quên thì log source mới không có tác dụng, và sẽ rất khó hiểu vì sao không có alert.
- Phải chắc chắn Manager lắng nghe syslog đúng cổng đã đặt trên pfSense, và tường lửa cho phép cổng đó. Sai cổng thì log sẽ không bao giờ tới nơi.
- Khi demo flood, em chặn thẳng IP máy tấn công trên pfSense cho nhanh. Trong mạng thực tế nên chặn theo nhánh mạng khác hoặc theo cổng cụ thể, không chặn toàn bộ TCP.
- Nên dùng `wazuh-logtest` trước. Dán dòng log gốc vào và xem rule nào được kích hoạt. Cách này tiết kiệm nhiều thời gian hơn việc restart Manager liên tục.
- Log nhiễu là phần làm em mệt nhất. Quá nhiều cảnh báo sai sẽ che mất cảnh báo thật. Chỉnh ngưỡng là việc chậm, nhưng thật sự quan trọng.
- Lần đầu thấy alert hiện lên trên Kibana, em thật sự rất vui. Mọi bước cài đặt phía trước đều thấy xứng đáng.
- Các địa chỉ IP trong repo là của lab em. Hãy đổi lại cho đúng với mạng của bạn.
- Chỉ tấn công trong lab của mình. Không dùng `hydra` hay `hping3` lên hệ thống thật khi chưa được phép.
- Đây là một lab nhỏ. Nó không phải là một SOC hoàn chỉnh.
