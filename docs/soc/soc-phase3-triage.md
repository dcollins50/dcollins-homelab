**Phase 3: Alert Triage Workflow**   |   April 19, 2026

**HOMELAB SOC**

Phase 3: Alert Triage Workflow

*Learning Exercise and Standard Operating Procedure*

| **Document Date** | April 19, 2026 |
| --- | --- |
| **Phase** | 3 of 5 |
| **Author** | Daniel Collins |
| **Prerequisite** | Phase 2 tuning complete, alert volume stable |
| **Estimated Time** | Ongoing (daily practice) |

# **1. Purpose and Learning Objectives**

> **Status:** Live

---



Alert triage is the core daily skill of a SOC analyst. It is the process of taking a list of alerts and systematically working through them to determine which ones represent real threats, which are false positives, and which need deeper investigation. This phase defines a repeatable triage process that mirrors what production SOC teams use.

| **WHY** | The difference between a SOC analyst and someone who has a SIEM is the triage process. A tool generates alerts. An analyst applies judgment to those alerts. Triage is where that judgment is exercised. Building this habit in a homelab environment, against real (though lower-stakes) data, is exactly the kind of practice that translates directly to employment. |
| --- | --- |

## **1.1 Learning Objectives**

- Apply a consistent severity-first triage process to incoming alerts

- Use Kibana to enrich alert context before making a disposition decision

- Distinguish between a true positive, a false positive, and an investigation-needed disposition

- Know when to escalate an alert to a formal case in DFIR-IRIS

- Document triage decisions in a way that is useful for future reference

## **1.2 Success Criteria**

This phase has no single completion point. It is a recurring practice. Success is defined as performing a triage review session at least twice per week during the homelab active period and maintaining a triage log that records your dispositions.

# **2. Background: The Triage Mindset**

## **2.1 What Triage Is Not**

Triage is not the same as incident response. Triage is the sorting step that happens before incident response. The goal of triage is to move quickly through a list of alerts and classify each one, not to fully investigate every alert in the list. Think of it like emergency room triage: the nurse's job is not to treat every patient on the spot. The job is to identify which patients need immediate attention and which can wait.

## **2.2 The Three Dispositions**

| **Disposition** | **Definition** | **Next Action** |
| --- | --- | --- |
| **TRUE POSITIVE** | The alert reflects real malicious or unauthorized activity. | Open a case in DFIR-IRIS immediately. Begin incident response. |
| **FALSE POSITIVE** | The alert fired but the activity is benign and expected. | Document it. Flag the rule for Phase 2 tuning review if it fires repeatedly. |
| **NEEDS INVESTIGATION** | The alert cannot be disposed of quickly. The activity is unclear and requires deeper analysis. | Open a case in DFIR-IRIS. Schedule time for investigation. Do not leave it in a limbo state. |

| **NOTE** | When in doubt, open a case. It is always better to investigate something that turns out to be benign than to dismiss something that turns out to be a breach. The cost of a false investigation is time. The cost of a missed breach is everything else. |
| --- | --- |

# **3. The Triage Process**

## **3.1 Overview**

A triage session follows five steps in order. Do not skip steps or reorder them. The sequence is designed so that each step builds on the information from the previous one.

| **Step** | **Name** | **Description** |
| --- | --- | --- |
| **1** | Severity Sort | Filter to high severity alerts first. Do not look at anything below level 7 until all high severity items have been reviewed. |
| **2** | Context Enrichment | For each alert, gather additional context: what agent, what time, what else was happening on that agent at the same time. |
| **3** | Pattern Check | Is this alert part of a pattern? Has this rule fired on this agent before? Is it happening in isolation or alongside other alerts? |
| **4** | Disposition | Make a decision: true positive, false positive, or needs investigation. |
| **5** | Documentation | Record your disposition and reasoning. For true positives and needs investigation, open a case in DFIR-IRIS. |

## **3.2 Step 1: Severity Sort**

Open the SIEM Baseline dashboard in Kibana. Set the time range to the last 24 hours. Apply a filter to show only alerts with rule.level greater than or equal to 7.

In the KQL search bar:

rule.level >= 7

Work through all high severity alerts before dropping the threshold. This ensures that genuinely dangerous events are not buried under medium severity noise.

## **3.3 Step 2: Context Enrichment**

For each alert that warrants investigation, click into the full alert document in Kibana and gather the following:

| **Context Item** | **Why It Matters** |
| --- | --- |
| agent.name and agent.ip | Identifies the affected host. Is this a high-value target like soc-stack or a lower-risk host like kalshi-mm? |
| @timestamp | When did it happen? During business hours, off hours, or during a known maintenance window? |
| rule.groups | What category of activity triggered this? Authentication failures and file integrity events have very different implications. |
| data.srcip or data.srcuser | Where did the activity originate? A known internal IP is different from an unknown external one. |
| full.log or data fields | The raw log entry that triggered the alert. Reading the original log is often the fastest way to understand what actually happened. |

After gathering context, run a secondary Kibana query to see all other alerts from the same agent in the same time window:

agent.name: "<agent_name>" AND @timestamp: [now-1h TO now]

Alerts that appear in clusters are more suspicious than isolated single events.

## **3.4 Step 3: Pattern Check**

Ask the following questions before making a disposition:

- Has this exact rule fired on this agent in the last 7 days? If yes, how many times?

- Is the source IP or user associated with any other alerts across the fleet?

- Is this rule firing on multiple agents simultaneously? Lateral movement often shows up this way.

- Does the timing of this alert correlate with any known scheduled jobs, backup processes, or maintenance activities on that host?

Run a cross-agent query to check for simultaneous activity:

rule.id: "<rule_id>" AND @timestamp: [now-15m TO now]

## **3.5 Step 4: Disposition**

With context and pattern information in hand, make your disposition. Apply the following decision logic:

| **Condition** | **Disposition** |
| --- | --- |
| Unknown source IP, unusual time, first occurrence, high severity | **TRUE POSITIVE** |
| Known internal source, expected time, recurring pattern, matches known-good behavior | **FALSE POSITIVE** |
| Unusual but not clearly malicious, incomplete context, first occurrence on a sensitive host | **NEEDS INVESTIGATION** |
| Multiple agents, same rule, same time window, no known maintenance activity | **TRUE POSITIVE** |

## **3.6 Step 5: Documentation**

Record every disposition in your triage log. The format does not need to be elaborate, but it does need to be consistent. A simple markdown log entry is sufficient:

## 2026-04-19 Triage Session

### Alert: Rule 5503 - User login failure (agent: pve-gateway)

Level: 5 | Count: 3 in 24h

Context: Failures from 10.0.0.199 (my workstation) at 08:14.

         Likely mistyped password during morning login.

Disposition: FALSE POSITIVE

Action: None. If count exceeds 10/day, add to Phase 2 threshold list.

For true positives and needs investigation dispositions, open a case in DFIR-IRIS (Phase 4 covers this) and record the case number in the triage log.

# **4. Triage Session Schedule**

Triage is only valuable if it is done consistently. A SIEM that is reviewed once a week will miss events that happened six days ago. The recommended schedule for this homelab during active development is:

| **Frequency** | **Scope** | **Time Required** |
| --- | --- | --- |
| Daily (recommended) | Last 24 hours, all agents, level 7 and above | 15 to 30 minutes |
| Weekly | Last 7 days, all agents, level 4 and above | 45 to 90 minutes |
| Monthly | Full month review, trend analysis, tuning log review | 2 to 3 hours |

| **NOTE** | Daily triage does not need to be comprehensive. Fifteen minutes scanning high severity alerts is enough to catch anything genuinely dangerous. The weekly session is where you go deeper. |
| --- | --- |

# **5. Verification**

| **Checkpoint** | **Status** |
| --- | --- |
| Triage log file created and at least one session documented | **PASS / FAIL** |
| All level 12 and above alerts from the past 7 days have a documented disposition | **PASS / FAIL** |
| At least one triage session completed using the 5-step process | **PASS / FAIL** |
| Any true positive or needs investigation alerts have DFIR-IRIS cases opened | **PASS / FAIL** |

# **6. Quick Reference: Triage Checklist**

Print or bookmark this checklist for use during each triage session.

|  | **Triage Step** |
| --- | --- |
| [ ] | Set Kibana time range to last 24 hours |
| [ ] | Apply filter: rule.level >= 12. Review all results before proceeding. |
| [ ] | Apply filter: rule.level >= 7. Review remaining medium-high alerts. |
| [ ] | For each alert: check agent, timestamp, source IP, and raw log |
| [ ] | Run cluster check: same rule, same agent, last 1 hour |
| [ ] | Run cross-agent check: same rule, all agents, last 15 minutes |
| [ ] | Assign disposition: True Positive, False Positive, or Needs Investigation |
| [ ] | Record disposition in triage log |
| [ ] | Open DFIR-IRIS case for any True Positive or Needs Investigation result |
| [ ] | Flag any False Positive that has fired more than 3 times this week for Phase 2 tuning review |


github.com/dcollins50/dcollins-homelab  |  Internal Documentation  |  Not for Distribution