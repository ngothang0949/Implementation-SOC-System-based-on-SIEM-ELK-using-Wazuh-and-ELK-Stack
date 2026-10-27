<a id="en"></a>

<div align="center">

# SOC Lab: Wazuh (SIEM/EDR) + ELK Stack

**🇬🇧 English** &nbsp;|&nbsp; [🇻🇳 Tiếng Việt](#vi)

</div>

---

## 1. Overview

This is a small SOC (Security Operations Center) lab. It uses **Wazuh** as the SIEM/EDR platform. It is integrated with the **ELK Stack** (Elasticsearch, Logstash, Kibana).

The lab was built in a 12-week internship (Jan – Apr 2026). It is a learning project, not a production system.

What the system can do:

- Collect logs from endpoints (Ubuntu, Windows) with Wazuh Agent.
- Collect logs from devices without agent (pfSense) with syslog.
- Analyze logs with decoders and rules, then create alerts.
- Store and search alerts in Elasticsearch.
- Show alerts on Kibana dashboards.
- Detect SSH brute-force and TCP flood with custom rules.

> **Note:** The detection rules only cover a few attacks (SSH and TCP flood). So please do not think this is a full SOC. It is a base that I want to grow later.

---

## 2. Architecture

The pipeline has 5 layers:

| Layer | Component | Job |
|---|---|---|
| Collection | Wazuh Agent / syslog | Collect logs from endpoints and network devices |
| Analysis | Wazuh Manager | Decode logs, match rules, create alerts with MITRE ATT&CK tags |
| Forwarding | Logstash | Send alerts to Elasticsearch |
| Storage | Elasticsearch | Index and search alerts |
| Visualization | Kibana | Dashboards and alerting |

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

### Why Wazuh + ELK?

- Wazuh is strong at security analysis. It has rules, decoders and MITRE mapping.
- ELK is strong at storing, searching and visualizing big data.
- ELK alone has no threat detection rules. Wazuh alone is less flexible for visualization. Together they fit well.

---

## 3. Lab Network

![SOC lab network diagram](images/network-diagram.png)

pfSense is the router and the firewall between three LAN branches. It also sends its logs to the Manager, because it cannot run a Wazuh Agent.

| Device | Role | IP address | Gateway |
|---|---|---|---|
| pfSense (WAN) | Router / firewall / syslog source | 10.0.2.15/24 | - |
| pfSense (LAN1) | Gateway of LAN1 | 192.168.10.1/24 | - |
| pfSense (LAN2) | Gateway of LAN2 | 192.168.20.1/24 | - |
| pfSense (LAN3) | Gateway of LAN3 | 192.168.30.1/24 | - |
| Wazuh Manager + ELK | SIEM server | 192.168.10.105/24 | 192.168.10.1 |
| Ubuntu | Wazuh Agent | 192.168.20.103/24 | 192.168.20.1 |
| Windows 10 | Wazuh Agent | 192.168.30.104/24 | 192.168.30.1 |

> **Note:** LAN1 is the "LAN manager". You can open the pfSense web interface from the Manager machine by using its gateway IP `192.168.10.1`. It is very convenient, so I did not need another machine to manage the firewall.

---

## 4. Versions

| Software | Version |
|---|---|
| Elasticsearch | 7.10.2 |
| Kibana | 7.10.2 |
| Logstash | 7.10.2 (`1:7.10.2-1`) |
| Wazuh Manager | 4.3.11-1 |
| Wazuh Agent | 4.3.11 |
| OS of server | Ubuntu (apt-based) |

> **Note:** Please keep the versions the same as above. The Wazuh agent and manager should match. Different versions may cause errors that are hard to find.

---

## 5. Installation (on the Manager machine)

All of ELK and Wazuh Manager run on one server (`192.168.10.105`).

### 5.1 Elasticsearch

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

Start it and test:

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

curl http://localhost:9200
```

### 5.2 Kibana

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

### 5.3 Logstash

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

> **Note:** Do not skip the `usermod` step. Without it Logstash cannot read the alert file, and nothing shows up in Elasticsearch. There is no big error, so it is easy to miss.

Check the pipeline syntax, then start Logstash:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

sudo systemctl enable logstash
sudo systemctl start logstash
```

### 5.4 Wazuh Manager

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

### 5.5 Check the data flow

```bash
# 1. Wazuh creates alerts?
ls -l /var/ossec/logs/alerts/alerts.json

# 2. Index exists in Elasticsearch?
curl http://localhost:9200/_cat/indices?v
```

You should see an index like `wazuh-alerts-YYYY.MM.dd`.

Then open Kibana at `http://<IP_SERVER>:5601` and create an index pattern:

- Index pattern: `wazuh-alerts-*`
- Time field: `@timestamp`

> **Note:** The first time I saw the alerts appear in Kibana, I was really happy. All the steps before it felt worth it.

---

## 6. Wazuh Agents

Install Wazuh Agent 4.3.11 on the Ubuntu and Windows machines. Point them to the Manager `192.168.10.105`.

### Extra log source for Windows (SSH)

Wazuh Agent 4.3.11 does **not** collect OpenSSH logs by default. Add this block at the end of `ossec.conf` on the Windows agent:

```xml
<localfile>
  <location>OpenSSH/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

> **Note:** Remember to restart the agent after editing `ossec.conf`. If not, the new log source does not work and you will wonder why there is no alert.

---

## 7. pfSense (agentless logs)

pfSense cannot run a Wazuh Agent. So its logs go to the Manager by syslog.

- Install the **syslog-ng** package on pfSense. It collects, sorts, stores and forwards logs. The output is cleaner than the default pfSense log.
- pfSense sends the logs to the Manager (default syslog port is 514, and syslog-ng forwards to port 5140 in this lab).
- pfSense also works as the firewall for the LAN branches.

> **Note:** Make sure the Manager is listening for syslog on the port you set in pfSense, and the port is allowed by the firewall. If the port does not match, logs never arrive.

---

## 8. Custom Detection Rules

All rules go into `/var/ossec/etc/rules/local_rules.xml` on the Manager. After editing:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

You can test a rule before using it with `wazuh-logtest`:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

### 8.1 Ubuntu: SSH monitoring (rule IDs 100100 – 100104)

| Rule ID | Level | Detects |
|---|---|---|
| 100100 | 5 | SSH login successful |
| 100101 | 7 | SSH login failed |
| 100102 | 8 | SSH login with invalid (non-existent) user |
| 100103 | 12 | SSH brute force (5 failed logins in 60s) |
| 100104 | 15 | SSH brute force **succeeded** (account compromised) |

How the brute-force detection works:

1. 5 failed logins in 60 seconds.
2. A successful login comes from the same source IP / user.
3. Alert: possible account compromised.

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

### 8.2 Windows 10: OpenSSH monitoring (rule IDs 100300 – 100305)

| Rule ID | Level | Detects |
|---|---|---|
| 100300 | 0 | Parent rule: SSH login failed from `sshd.exe` (Security log) |
| 100301 | 7 | SSH wrong password |
| 100302 | 7 | SSH wrong username (user does not exist) |
| 100303 | 12 | SSH brute force (5 failed logins in 60s, same user) |
| 100304 | 4 | SSH login successful |
| 100305 | 15 | Suspected brute force **success** (100303 + 100304, same user) |

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

### 8.3 pfSense: flood detection (rule IDs 100500 – 100503)

The goal is to detect flood attacks into the management network (LAN1).

| Rule ID | Level | Detects |
|---|---|---|
| 100500 | 0 | Base rule for pfSense events |
| 100501 | 7 | Firewall **block** event |
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

> **Note:** In the demo I block the attacker IP on pfSense directly, because it is fast to test. In a real network you should block by other subnets, or by a specific port, not all TCP. Blocking everything is too strong, and it is only OK for a lab.

---

## 9. How to Test

Use the Manager machine (or a Kali machine) as the attacker. After each test, check the alerts on Kibana.

| Test | How | Expected rule |
|---|---|---|
| Valid SSH login | `ssh user@<target>` with the right password | 100100 (Ubuntu) / 100304 (Windows) |
| Wrong password | `ssh user@<target>` with a wrong password | 100101 / 100301 |
| Wrong username | `ssh fakeuser@<target>` | 100102 / 100302 |
| SSH brute force | `hydra` with a wordlist | 100103 / 100303 |
| Brute force success | Fail 5 times, then login right (6th try) or use `hydra` with the correct password in the list | 100104 / 100305 |
| TCP flood | Attacker in LAN3 (`192.168.30.0/24`) uses `hping3` against the Manager | 100501 → 100502 → 100503 |

Example commands:

```bash
# SSH brute force (example)
hydra -l <username> -P passwords.txt ssh://<target_ip>

# TCP flood (example)
sudo hping3 -S --flood -p <port> <target_ip>
```

> **Note:** Only attack your own lab. Please do not use these commands on any real system without permission.

**Tip:** use `wazuh-logtest` first. Paste the raw log line and see which rule fires. It saves a lot of time before you restart the Manager again and again.

---

## 10. Results

- All custom rules work. They detect valid, invalid and brute-force access.
- Kibana shows all the alerts from the custom rules.
- The full pipeline runs: Agent / syslog → Manager → Logstash → Elasticsearch → Kibana.
- Dashboards and basic alerts (email / Telegram) are set up on Kibana.
- The system was checked against MITRE ATT&CK, ISO and NIST.

**Conclusion:** the model works well with the new rules.

---

## 11. Limitations and Future Work

Still not done:

- Only basic SSH attacks are covered. HTTP/HTTPS, RDP, web apps, lateral movement and privilege escalation are not monitored well yet.
- Alerts only notify (email, Telegram). **Wazuh Active Response** is not used yet, so the system cannot block IPs or isolate endpoints automatically.
- There is still noisy log. Rules and thresholds need tuning for a real environment.
- ISO/IEC 27001 and NIST 800-53 are not fully satisfied. More rules are needed to map the missing controls.

> **Note:** Noisy logs were the most annoying part for me. Too many false alerts make real ones hard to see. Tuning thresholds is slow work, but it is really important.

---

## 12. What I Learned

- How a real SOC is designed, built and run.
- How to read documentation and apply it, even when it is hard.
- How to write rules and correlate events.
- How to debug problems: installation errors, agent connection, log pipeline.
- How to work in a team and report progress every week.

---

<div align="right">

[⬆ Back to top](#en) &nbsp;|&nbsp; [🇻🇳 Chuyển sang Tiếng Việt](#vi)

</div>

<br><br>

---
---

<a id="vi"></a>

<div align="center">

# Lab SOC: Wazuh (SIEM/EDR) + ELK Stack

[🇬🇧 English](#en) &nbsp;|&nbsp; **🇻🇳 Tiếng Việt**

</div>

---

## 1. Giới thiệu

Đây là một lab SOC (Security Operations Center) nhỏ. Hệ thống dùng **Wazuh** làm nền tảng SIEM/EDR. Wazuh được tích hợp với **ELK Stack** (Elasticsearch, Logstash, Kibana).

Lab được xây dựng trong 12 tuần thực tập (01 – 04/2026). Đây là đề tài học tập, chưa phải hệ thống chạy thực tế.

Hệ thống làm được những việc sau:

- Thu thập log từ endpoint (Ubuntu, Windows) bằng Wazuh Agent.
- Thu thập log từ thiết bị không cài được agent (pfSense) qua syslog.
- Phân tích log bằng decoder và rule, rồi sinh cảnh báo.
- Lưu trữ và truy vấn cảnh báo trong Elasticsearch.
- Hiển thị cảnh báo trên dashboard Kibana.
- Phát hiện SSH brute-force và TCP flood bằng rule tự viết.

> **Lưu ý:** Rule hiện chỉ bao phủ vài loại tấn công (SSH và TCP flood). Vì vậy đừng nghĩ đây là một SOC hoàn chỉnh. Nó chỉ là nền móng để nhóm tiếp tục phát triển sau này.

---

## 2. Kiến trúc

Pipeline gồm 5 tầng:

| Tầng | Thành phần | Nhiệm vụ |
|---|---|---|
| Thu thập | Wazuh Agent / syslog | Thu log từ endpoint và thiết bị mạng |
| Phân tích | Wazuh Manager | Giải mã log, so khớp rule, sinh cảnh báo kèm MITRE ATT&CK |
| Chuyển tiếp | Logstash | Đẩy cảnh báo sang Elasticsearch |
| Lưu trữ | Elasticsearch | Lập chỉ mục và tìm kiếm cảnh báo |
| Trực quan hóa | Kibana | Dashboard và cảnh báo |

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

### Vì sao kết hợp Wazuh + ELK?

- Wazuh mạnh về phân tích an ninh. Có sẵn rule, decoder và ánh xạ MITRE.
- ELK mạnh về lưu trữ, tìm kiếm và trực quan hóa dữ liệu lớn.
- ELK đứng một mình thì không có rule phát hiện mối đe dọa. Wazuh đứng một mình thì trực quan hóa kém linh hoạt hơn. Kết hợp lại thì bổ sung cho nhau.

---

## 3. Sơ đồ mạng lab

![Sơ đồ mạng lab SOC](images/network-diagram.png)

pfSense vừa là bộ định tuyến, vừa là tường lửa giữa ba nhánh LAN. Ngoài ra pfSense gửi log về Manager, vì nó không cài được Wazuh Agent.

| Thiết bị | Vai trò | Địa chỉ IP | Gateway |
|---|---|---|---|
| pfSense (WAN) | Router / tường lửa / nguồn syslog | 10.0.2.15/24 | - |
| pfSense (LAN1) | Gateway của LAN1 | 192.168.10.1/24 | - |
| pfSense (LAN2) | Gateway của LAN2 | 192.168.20.1/24 | - |
| pfSense (LAN3) | Gateway của LAN3 | 192.168.30.1/24 | - |
| Wazuh Manager + ELK | Máy chủ SIEM | 192.168.10.105/24 | 192.168.10.1 |
| Ubuntu | Wazuh Agent | 192.168.20.103/24 | 192.168.20.1 |
| Windows 10 | Wazuh Agent | 192.168.30.104/24 | 192.168.30.1 |

> **Lưu ý:** LAN1 là "LAN manager". Có thể mở giao diện web của pfSense ngay từ máy Manager, thông qua IP gateway `192.168.10.1`. Cách này rất tiện, không cần thêm máy khác để quản lý tường lửa.

---

## 4. Phiên bản

| Phần mềm | Phiên bản |
|---|---|
| Elasticsearch | 7.10.2 |
| Kibana | 7.10.2 |
| Logstash | 7.10.2 (`1:7.10.2-1`) |
| Wazuh Manager | 4.3.11-1 |
| Wazuh Agent | 4.3.11 |
| Hệ điều hành máy chủ | Ubuntu (dùng apt) |

> **Lưu ý:** Hãy giữ đúng các phiên bản như trên. Agent và Manager của Wazuh nên cùng phiên bản. Khác phiên bản có thể gây lỗi rất khó tìm.

---

## 5. Cài đặt (trên máy Manager)

Toàn bộ ELK và Wazuh Manager chạy trên một máy chủ (`192.168.10.105`).

### 5.1 Elasticsearch

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

Sửa file `/etc/elasticsearch/elasticsearch.yml` (chạy single node):

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

### 5.2 Kibana

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

### 5.3 Logstash

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

> **Lưu ý:** Đừng bỏ qua bước `usermod`. Thiếu bước này Logstash không đọc được file alert, và Elasticsearch sẽ không có dữ liệu. Hệ thống không báo lỗi lớn nên rất dễ bỏ sót.

Kiểm tra cú pháp pipeline rồi khởi động Logstash:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

sudo systemctl enable logstash
sudo systemctl start logstash
```

### 5.4 Wazuh Manager

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

### 5.5 Kiểm tra luồng dữ liệu

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

> **Lưu ý:** Lần đầu thấy alert hiện lên trên Kibana, mình thật sự rất vui. Mọi bước phía trước đều thấy xứng đáng.

---

## 6. Wazuh Agent

Cài Wazuh Agent 4.3.11 trên máy Ubuntu và Windows. Trỏ agent về Manager `192.168.10.105`.

### Thêm log source cho Windows (SSH)

Wazuh Agent 4.3.11 **không** thu log OpenSSH theo mặc định. Thêm đoạn sau vào cuối file `ossec.conf` của agent Windows:

```xml
<localfile>
  <location>OpenSSH/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

> **Lưu ý:** Nhớ restart agent sau khi sửa `ossec.conf`. Nếu quên thì log source mới không có tác dụng, và sẽ rất khó hiểu vì sao không có alert.

---

## 7. pfSense (log không dùng agent)

pfSense không cài được Wazuh Agent. Vì vậy log của nó được gửi về Manager qua syslog.

- Cài gói **syslog-ng** trên pfSense. Gói này thu thập, phân loại, lưu trữ và chuyển tiếp log. Log đầu ra dễ đọc và chuẩn hóa tốt hơn log mặc định.
- pfSense gửi log về Manager (cổng syslog mặc định là 514, và trong lab này syslog-ng chuyển tiếp qua cổng 5140).
- pfSense cũng là tường lửa bảo vệ các nhánh LAN.

> **Lưu ý:** Phải chắc chắn Manager đang lắng nghe syslog đúng cổng đã cấu hình trên pfSense, và cổng đó được tường lửa cho phép. Sai cổng thì log sẽ không bao giờ tới nơi.

---

## 8. Rule phát hiện tự viết

Tất cả rule được thêm vào `/var/ossec/etc/rules/local_rules.xml` trên Manager. Sau khi sửa:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

Có thể test rule trước bằng `wazuh-logtest`:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

### 8.1 Ubuntu: giám sát SSH (rule ID 100100 – 100104)

| Rule ID | Level | Phát hiện |
|---|---|---|
| 100100 | 5 | Đăng nhập SSH thành công |
| 100101 | 7 | Đăng nhập SSH thất bại |
| 100102 | 8 | Đăng nhập SSH bằng user không tồn tại |
| 100103 | 12 | SSH brute force (5 lần thất bại trong 60s) |
| 100104 | 15 | SSH brute force **thành công** (tài khoản bị chiếm) |

Cách phát hiện brute force:

1. Đăng nhập thất bại 5 lần trong 60 giây.
2. Có đăng nhập thành công từ cùng IP nguồn / cùng user.
3. Gửi alert nghi ngờ tài khoản đã bị chiếm.

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

### 8.2 Windows 10: giám sát OpenSSH (rule ID 100300 – 100305)

| Rule ID | Level | Phát hiện |
|---|---|---|
| 100300 | 0 | Rule cha: SSH thất bại từ `sshd.exe` (Security log) |
| 100301 | 7 | SSH sai password |
| 100302 | 7 | SSH sai username (user không tồn tại) |
| 100303 | 12 | SSH brute force (5 lần thất bại trong 60s, cùng user) |
| 100304 | 4 | Đăng nhập SSH thành công |
| 100305 | 15 | Nghi ngờ brute force **thành công** (100303 + 100304, cùng user) |

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

### 8.3 pfSense: phát hiện flood (rule ID 100500 – 100503)

Mục tiêu là phát hiện các cuộc tấn công flood vào nhánh mạng quản lý (LAN1).

| Rule ID | Level | Phát hiện |
|---|---|---|
| 100500 | 0 | Rule gốc cho event của pfSense |
| 100501 | 7 | Event tường lửa **chặn** kết nối |
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

> **Lưu ý:** Khi demo, nhóm chặn thẳng IP máy tấn công trên pfSense cho nhanh. Trong mạng thực tế nên chặn theo nhánh mạng khác, hoặc chỉ định cổng cần bảo vệ, thay vì chặn toàn bộ TCP. Chặn hết như vậy quá mạnh tay, chỉ phù hợp với lab.

---

## 9. Cách kiểm thử

Dùng máy Manager (hoặc máy Kali) làm máy tấn công. Sau mỗi lần test, xem alert trên Kibana.

| Test | Cách làm | Rule mong đợi |
|---|---|---|
| SSH đúng | `ssh user@<target>` với đúng password | 100100 (Ubuntu) / 100304 (Windows) |
| Sai password | `ssh user@<target>` với password sai | 100101 / 100301 |
| Sai username | `ssh fakeuser@<target>` | 100102 / 100302 |
| SSH brute force | `hydra` với wordlist | 100103 / 100303 |
| Brute force thành công | Sai 5 lần rồi đăng nhập đúng (lần thứ 6), hoặc dùng `hydra` với wordlist có password đúng | 100104 / 100305 |
| TCP flood | Máy tấn công ở LAN3 (`192.168.30.0/24`) dùng `hping3` vào máy Manager | 100501 → 100502 → 100503 |

Lệnh ví dụ:

```bash
# SSH brute force (ví dụ)
hydra -l <username> -P passwords.txt ssh://<target_ip>

# TCP flood (ví dụ)
sudo hping3 -S --flood -p <port> <target_ip>
```

> **Lưu ý:** Chỉ tấn công trong lab của mình. Đừng dùng các lệnh này lên hệ thống thật khi chưa được cho phép.

**Mẹo:** nên dùng `wazuh-logtest` trước. Dán dòng log gốc vào và xem rule nào được kích hoạt. Cách này tiết kiệm rất nhiều thời gian, đỡ phải restart Manager nhiều lần.

---

## 10. Kết quả

- Tất cả rule tự viết đều hoạt động. Phát hiện được truy cập hợp lệ, không hợp lệ và brute force.
- Kibana thấy được toàn bộ alert sinh ra từ các rule này.
- Pipeline chạy đủ: Agent / syslog → Manager → Logstash → Elasticsearch → Kibana.
- Đã có dashboard và cảnh báo cơ bản (email / Telegram) trên Kibana.
- Hệ thống đã được đánh giá theo MITRE ATT&CK, ISO và NIST.

**Kết luận:** mô hình hoạt động bình thường với các rule mới.

---

## 11. Hạn chế và hướng phát triển

Những phần chưa làm được:

- Mới chỉ tập trung vào tấn công SSH cơ bản. Chưa giám sát tốt HTTP/HTTPS, RDP, ứng dụng web, tấn công chiều ngang và thăng quyền.
- Cảnh báo mới dừng ở thông báo (email, Telegram). Chưa tích hợp **Active Response** của Wazuh, nên hệ thống chưa tự chặn IP hay cô lập endpoint được.
- Vẫn còn log nhiễu. Cần tinh chỉnh rule và ngưỡng cảnh báo cho hợp với môi trường thực tế.
- ISO/IEC 27001 và NIST 800-53 chưa đáp ứng đầy đủ. Cần thêm rule ánh xạ với các control còn thiếu.

> **Lưu ý:** Log nhiễu là phần làm mình mệt nhất. Quá nhiều cảnh báo sai thì cảnh báo thật rất khó thấy. Chỉnh ngưỡng là việc chậm, nhưng thật sự quan trọng.

---

## 12. Bài học rút ra

- Hiểu cách một hệ thống SOC thực tế được thiết kế, xây dựng và vận hành.
- Biết đọc tài liệu kỹ thuật và áp dụng vào thực tế, dù đôi lúc khá khó.
- Biết viết rule và tương quan sự kiện (correlation).
- Biết xử lý lỗi: lỗi cài đặt, lỗi kết nối agent, lỗi pipeline log.
- Rèn kỹ năng làm việc nhóm và báo cáo tiến độ hằng tuần.

---

<div align="right">

[⬆ Về đầu trang](#vi) &nbsp;|&nbsp; [🇬🇧 Switch to English](#en)

</div>
