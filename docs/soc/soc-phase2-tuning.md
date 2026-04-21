**Phase 2: Noise Reduction and Rule Tuning**   |   April 19, 2026

**HOMELAB SOC**

Phase 2: Noise Reduction and Rule Tuning

*Learning Exercise and Standard Operating Procedure*

| **Document Date** | April 19, 2026 |
| --- | --- |
| **Phase** | 2 of 5 |
| **Author** | Daniel Collins |
| **Prerequisite** | Phase 1 baseline document completed |
| **Estimated Time** | 4 to 8 hours (ongoing) |

# **1. Purpose and Learning Objectives**

> **Status:** Live

---



Phase 2 takes the ranked baseline list from Phase 1 and turns it into a deliberate tuning decision for every noisy rule. The goal is not silence. The goal is signal. A well-tuned SIEM produces alerts that are worth reading. An untuned SIEM produces so many alerts that analysts stop reading them. Alert fatigue is a security failure, not just an operational inconvenience.

| **WHY** | The 2013 Target breach is a frequently cited example of alert fatigue as a root cause. The FireEye system detected the malware and generated alerts. The SOC team received so many alerts that they did not investigate the ones that mattered. The lesson is not that more alerts are safer. The lesson is that fewer, higher-quality alerts save companies. |
| --- | --- |

## **1.1 Learning Objectives**

- Understand the Wazuh rule hierarchy and how custom rules override built-in rules

- Learn the three tuning decisions available for any noisy rule: suppress, threshold, or accept

- Write custom Wazuh rules in XML that modify built-in rule behavior

- Maintain a tuning decision log that explains every change for future reference

- Understand the difference between suppressing a rule entirely and adding conditions to it

## **1.2 Success Criteria**

This phase is complete when every rule in your Phase 1 top 20 list has a documented tuning decision, custom rules have been written and tested for all suppress and threshold decisions, and alert volume has measurably decreased from the baseline level.

# **2. Background: Wazuh Rule Architecture**

## **2.1 How Rules Are Organized**

Wazuh rules are XML files stored in two locations. Built-in rules live in /var/ossec/ruleset/rules/ and should never be edited directly. Editing them means your changes get overwritten on the next Wazuh update. Custom rules live in /var/ossec/etc/rules/ and persist across updates. All tuning work happens in the custom rules directory.

The main custom rules file is local_rules.xml. You can also create additional files in the same directory and Wazuh will load them all.

## **2.2 Rule Override Mechanics**

When a custom rule shares the same rule ID as a built-in rule, the custom rule wins. This is how you modify built-in behavior. You can also create a rule that uses the overwrite attribute to completely replace a built-in rule, or you can create a new rule with a higher ID that references the built-in rule and modifies its behavior.

Custom rule IDs must be in the range 100000 to 120000. IDs below 100000 are reserved for built-in rules.

## **2.3 The Three Tuning Decisions**

| **Decision** | **When to Use It** | **How to Implement It** |
| --- | --- | --- |
| Suppress | The rule fires on events that are always expected in your environment and have no security value. | Write a custom rule that matches the same condition and sets the alert level to 0. Level 0 alerts are not stored. |
| Threshold | The rule fires on events that are worth knowing about only when they happen more than N times in a window. | Use the frequency and timeframe attributes in your custom rule to require a count before alerting. |
| Accept | The rule fires on real security signals that you want to keep. Volume is acceptable or the rule is high severity. | Document the decision and move on. No rule change needed. |

# **3. Prerequisites**

- Phase 1 baseline document with top 20 rules ranked by alert count

- SSH access to wazuh-manager (VM 601, 10.0.10.11)

- Sudo access on wazuh-manager

- Understanding of basic XML syntax

# **4. Procedure**

## **4.1 Set Up the Tuning Decision Log**

Before making any changes, create a tuning decision log. This is a simple text or markdown file that records every change you make, why you made it, and the date. Future-you will thank present-you for this.

Create the log file on wazuh-manager:

sudo nano /var/ossec/etc/rules/tuning_log.md

Use this structure for each entry:

## Rule 5706 - SSH Insecure Connection Attempt

Date: 2026-04-19

Decision: Suppress

Reason: Fired 4,200 times in 7 days, all from internal management

         workstation 10.0.0.199. Known expected behavior from

         automated monitoring scripts.

Custom Rule ID: 100001

## **4.2 Open the Custom Rules File**

sudo nano /var/ossec/etc/rules/local_rules.xml

The file should already contain the opening and closing group tags. All your custom rules go inside this group:

<group name="local,">

  <!-- Custom rules go here -->

</group>

## **4.3 Write Suppression Rules**

A suppression rule fires on the same condition as the original rule but sets the level to 0. The overwrite attribute tells Wazuh to replace the original rule rather than add a new one.

Example: Suppress rule 5706 entirely:

<rule id="5706" level="0" overwrite="yes">

  <if_sid>5700</if_sid>

  <description>SSH insecure connection (suppressed)</description>

</rule>

| **WARNING** | Using overwrite suppresses the rule for ALL agents. If you want to suppress only for a specific agent or source IP, use a new rule ID with a match or srcip condition instead of the overwrite approach. |
| --- | --- |

Example: Suppress a rule only for a specific agent:

<rule id="100001" level="0">

  <if_sid>5706</if_sid>

  <match>agent_name: kalshi-mm</match>

  <description>SSH suppressed for kalshi-mm (expected)</description>

</rule>

## **4.4 Write Threshold Rules**

A threshold rule requires a condition to occur N times within a time window before alerting. This is useful for rules that fire on events that are only suspicious in bulk, such as authentication failures.

Example: Only alert on SSH failures if there are 10 or more in 60 seconds from the same source:

<rule id="100002" level="10" frequency="10" timeframe="60">

  <if_matched_sid>5710</if_matched_sid>

  <same_source_ip />

  <description>SSH brute force: 10 failures in 60s from same IP</description>

  <group>authentication_failures,pci_dss_10.2.4,</group>

</rule>

The same_source_ip tag means the counter resets per source IP. Without it, failures from 10 different IPs in 60 seconds would also trigger the rule.

## **4.5 Test Each Rule Change**

After adding any rule, test the configuration before restarting the manager:

/var/ossec/bin/wazuh-logtest

Then validate the full config:

sudo /var/ossec/bin/ossec-logtest -t

If no errors appear, restart the manager:

sudo systemctl restart wazuh-manager

After restarting, monitor the log for any rule load errors:

sudo tail -f /var/ossec/logs/ossec.log | grep -i 'error\|rule'

## **4.6 Work Through the Baseline List**

Starting from rule number 1 on your Phase 1 list (highest volume), work through each rule in order. For each one:

- Read the rule description and understand what it detects

- Check a sample of the actual alert documents in Kibana to understand what events are triggering it

- Make your tuning decision: suppress, threshold, or accept

- If suppress or threshold: write the custom rule, test it, and restart the manager

- Record the decision in the tuning log

- Move to the next rule

| **NOTE** | Work through the list in order but do not rush. Each decision deserves genuine thought. A rule you suppress today is a detection you no longer have tomorrow. Make sure you understand what you are turning off and why. |
| --- | --- |

## **4.7 Validate the Reduction**

After working through the top 20 list, return to your Phase 1 baseline dashboard. Set the time range to the 7 days following your tuning work and compare alert volume to the pre-tuning baseline. Document the before and after counts in your tuning log.

A well-tuned initial pass typically reduces alert volume by 40 to 70 percent without removing any genuine detection capability.

# **5. Verification**

| **Checkpoint** | **Status** |
| --- | --- |
| All 20 baseline rules have a documented tuning decision | **PASS / FAIL** |
| Custom rules file passes ossec-logtest validation | **PASS / FAIL** |
| Wazuh manager restarted successfully after each rule change | **PASS / FAIL** |
| Alert volume measurably reduced from Phase 1 baseline | **PASS / FAIL** |
| No level 12 or above rules suppressed without documented investigation | **PASS / FAIL** |
| Tuning decision log saved with entry for each changed rule | **PASS / FAIL** |

# **6. Troubleshooting**

## **Wazuh manager fails to start after rule change**

The XML is likely malformed. Check syntax with:

sudo xmllint --noout /var/ossec/etc/rules/local_rules.xml

Common mistakes are unclosed tags, missing quotes on attribute values, and special characters like ampersands that need to be escaped as &amp;

## **Suppressed rule is still generating alerts**

Confirm the custom rule was loaded by checking the ossec.log after restart:

sudo grep 'Loaded rule' /var/ossec/logs/ossec.log | grep '100001'

If the rule is not in the log, the file may not be in the correct directory or the group tags may be malformed.

## **Alert volume has not decreased after tuning**

Check whether the rules you targeted are actually the ones firing. The baseline dashboard may have changed over the 7-day comparison window as new activity started. Re-run the top 20 query against the post-tuning period to see if different rules are now at the top of the list.


github.com/dcollins50/dcollins-homelab  |  Internal Documentation  |  Not for Distribution